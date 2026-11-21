# MCP 三次复审修复计划（Plan）

- **基准提交**：
  - `cffacd34afb4219fa1562c319cce9d51b54f8946`（`feat: Antigravity MCP`）
  - `3081dfcf29c1ec2eed7d578aa42089a8d3886c90`（`refactor: not only Antigravity`）
  - `91351eaf87236fe9147401386bf433e5f5286e0d`（`fix: review`）
  - `cca984076abb440f26473732d36b075b02d604db`（`fix: review again`）
- **改造目标**：修复三次复审中发现的终态任务/零待翻页任务调用 `next_translation_batch` 抛 `KeyError`、分段末尾无字页/错误页未封口批次导致跨跳过页合批、`retranslate_json` 缺少 `load_text_parse_failures` 回写保护与预处理钩子、异步收尾线程与新任务竞态等问题，并补齐对应回归测试。
- **核心原则**：保持对外 MCP 工具协议与 0-API 硬隔离不变，所有修复向后兼容现有多 Harness 客户端与 Skill 工作流。

---

## 一、三次复审问题清单

| 编号 | 优先级 | 类别 | 涉及文件与位置 | 问题描述 |
| --- | --- | --- | --- | --- |
| R3-1 | 高 | 状态机与协议回归 | `manga_translator/mcp/job_manager.py:1739-1755`、`1994-2002`、`2029-2035`；`manga_translator/mcp/skill_template.md:32-33`、`66-67` | `91351eaf`（P0-2）在任务完成后将 `job_id` 从 `self._jobs` 弹出并存入 `self._completed_job_summaries`，且 `create_translation_job` 在 `pending_pages == 0`（全部跳过）时仅写入 `_completed_job_summaries` 不写入 `_jobs`。但 `McpJobManager.next_translation_batch` 仍调用只查 `_jobs` 的 `_get_job(job_id)`，导致：① `pending_pages == 0` 时按 Skill 指引调 `next_translation_batch` 必抛 `KeyError`；② `finalizing_render` 等待超时后后台落盘完成、客户端再次轮询 `next_translation_batch` 抛 `KeyError`；③ `return_next=False` 或多 Worker 并发收尾时后续轮询抛 `KeyError`。 |
| R3-2 | 中 | 切批边界与内存清理 | `manga_translator/mcp/job_manager.py:797-839`、`960-965` | 断点续传按跳过页将待翻页切分为 `self._segments`。但在 `_run_gpu_worker` 的 `preprocess` 分支中，`at_segment_end` 触发 `_seal_building_batch_locked()` 的逻辑仅在 `else:`（有文本区域）和 `except Exception:` 中执行，遗漏了 `if getattr(ctx, "translation_error", None):` 与 `elif not getattr(ctx, "text_regions", None):`（无字插画页）。若某分段前序页已在 `_building_batch_pages` 中而分段末尾页恰好为无字页或错误页，当前批次不会封口，导致下一分段（跳过页之后）的页面被混入同一批次。此外第 801、832、964 行异常清理 `_cleanup_context_memory` 后未置 `page.ctx = None`。 |
| R3-3 | 低 | JSON 重翻一致性 | `manga_translator/mcp/batch_adapter.py:303-363`、`954-960` | 对比主流程 `manga_translator.py:4330-4342` 与 `4446-4458`：① `preprocess_page(..., retranslate_json=True)` 记录了 `ctx.load_text_parse_failures = parse_failures`，但 `render_and_save_page` 回写 `_save_text_to_file` 时未检查该计数，若原 JSON 存在解析失败区域，回写会覆盖丢失原区域；② `preprocess_page(..., retranslate_json=True)` 未调用 `_prepare_loaded_regions`（补齐缺失 `font_size`）、`ctx.skip_font_scaling = _skip_scale` 与 `_apply_pre_dictionary_to_regions`（译前字典）。 |
| R3-4 | 低 | 异步收尾竞态 | `manga_translator/mcp/job_manager.py:549-567`、`1633-1647`、`1746-1761`；`desktop_qt_ui/services/mcp_service.py:373-380` | `_finalize_job_locked()` 先 `self._finished_event.set()` 唤醒调用方返回 `done=True`，再由后台 `mcp-finish` 线程调用 `_handle_job_finished`。若客户端收到 `done=True` 后立即创建新任务 `job2`，延迟运行的 `_handle_job_finished(job1)` 仍会无条件调用 `_schedule_idle_cleanup_locked()`，且 `McpService._on_job_finished` 会用 `job1` 的终态覆盖 `job2` 的 UI 进度卡片。此外 `cancel()` 在锁外重复调用了 `_preview_pool.shutdown`。 |
| R3-5 | 低 | 文案与安全配置 | `desktop_qt_ui/locales/zh_CN.json:1600`；`manga_translator/mcp/server_app.py:71-77` | ① `zh_CN.json` 中 `"MCP Job Status Summary"` 将 `{pending}`（实际为 `pending_pages`，即任务非跳过的待翻总页数）译为 `"待渲染: {pending}"`，全部渲染完成后仍显示总待翻页数，易引起误解；② `server_app.py` 的 `allowed_origins` 包含 `http://[::1]:*`、`https://127.0.0.1:*`、`https://localhost:*`，缺少对称的 `https://[::1]:*`。 |

---

## 二、分项修复方案

### 2.1 `next_translation_batch` 与 `submit_translation_batch` 支持已归档终态任务（R3-1）

1. **新增终态 Payload 构造辅助方法**：在 `McpJobManager` 中新增 `_build_done_payload_from_summary_locked(self, job_id: str, cached: Dict[str, Any]) -> BatchPayload`：
   - 从 `cached`（即 `JobStatusResponse.model_dump()` 快照）提取 9 个进度计数字段构造 `JobProgressSummary`；
   - 组装与 `McpTranslationJob._build_completion_summary_locked()` 结构一致的 `summary` 字典（包含 `job_id`、`status`、`output_dir`、`output_paths`、`failed_details` 及各进度计数）；
   - 返回 `BatchPayload(done=True, job_id=job_id, progress=progress, summary=summary)`。
2. **`McpJobManager.next_translation_batch()` 回退查 `_completed_job_summaries`**：
   - 在 `with self._lock:` 下先解析 `target_id = job_id or self._active_job_id or self._latest_job_id`；
   - 若 `target_id` 在 `self._jobs` 中，沿用 `job.next_translation_batch(worker_id=worker_id)`；
   - 若 `target_id` 不在 `self._jobs` 但在 `self._completed_job_summaries` 中，直接返回 `(self._build_done_payload_from_summary_locked(target_id, cached), [])`；
   - 否则才抛出 `KeyError(f"Translation job not found: {job_id or '<active>'}")`。
3. **`McpJobManager.submit_translation_batch()` 终态友好报错**：
   - 若 `target_id` 已不在 `self._jobs` 但在 `self._completed_job_summaries` 中，返回 `(SubmitBatchResponse(status="error", job_id=target_id, batch_id=batch_id, done=True, summary=dict(cached), message=f"Job '{target_id}' is already {cached.get('status', 'completed')}."), [])`，避免客户端重试提交时遭遇未捕获的 `KeyError`。

### 2.2 修复分段末尾无字页/错误页的批次封口与失败 `ctx` 置空（R3-2）

1. **统一分段封口检查**：在 `McpTranslationJob._run_gpu_worker()` 的 `if kind == "preprocess":` 持锁块（`job_manager.py:797-820`）中，将批次封口判断移出 `else:` 分支，置于 `if / elif / else` 之后：

```python
if getattr(ctx, "translation_error", None):
    page.failed = True
    page.error = str(ctx.translation_error)
    self._inflight_page_indices.discard(p_idx)
    self.translator._cleanup_context_memory(ctx, keep_result=False)
    page.ctx = None
elif not getattr(ctx, "text_regions", None):
    page.is_textless = True
    page.committed = True
    page.inpaint_done = True
    page.render_queued = True
    self._render_queue.append(p_idx)
else:
    self._building_batch_pages.append(p_idx)

if self._building_batch_pages and (
    len(self._building_batch_pages) >= self.batch_size
    or at_segment_end
    or len(self._inflight_page_indices) >= self.max_inflight_pages
):
    self._seal_building_batch_locked()
```

2. **失败分支立即置空 `page.ctx = None`**：
   - 在 `preprocess` 的 `ctx.translation_error` 分支（第 801 行）、`except Exception:` 分支（第 832 行）以及 `_run_render_worker` 的 `except Exception:` 分支（第 964 行）调用 `_cleanup_context_memory` 后，统一补上 `page.ctx = None`。

### 2.3 补齐 `retranslate_json` 区域准备、译前字典与解析失败保护（R3-3）

1. **`preprocess_page(..., retranslate_json=True)`（`batch_adapter.py:354-363`）**：
   - 调用 `translator._prepare_loaded_regions(loaded_regions, use_text_as_translation=False)` 补齐缺失的 `font_size`；
   - 清空旧译文（`region.translation = ""` 等）并赋值给 `ctx.text_regions = loaded_regions`、`ctx.skip_font_scaling = bool(_skip_scale)`；
   - 调用 `translator._apply_pre_dictionary_to_regions(ctx)` 使 `pre_dict` 在 JSON 重翻模式下同样生效。
2. **`render_and_save_page()`（`batch_adapter.py:954-960`）**：
   - 回写 JSON 前检查 `parse_failures = int(getattr(ctx, "load_text_parse_failures", 0) or 0)`；
   - 若 `parse_failures > 0`，记录 `logger.error(...)` 并跳过 `translator._save_text_to_file(image_name, ctx, config)`，防止覆盖损坏原工程 JSON。

### 2.4 收敛异步 `_finish_cleanup` 与新任务的竞态窗口（R3-4）

1. **`McpJobManager._handle_job_finished()`（`job_manager.py:1746-1761`）**：
   - 在 `with self._lock:` 中检查当前是否已有新任务正在创建或运行（`is_idle = not self._creating_job and self._active_job_id is None`）；
   - 仅当 `is_idle` 为 `True` 时才调用 `self._schedule_idle_cleanup_locked()`。
2. **`McpService._on_job_finished()`（`mcp_service.py:373-380`）**：
   - 若 `not self.has_active_job()`（或当前活跃任务即刚完成的 `summary.get("job_id")`），才发射 `self.job_progress_changed.emit(summary)`；`self.job_finished.emit(summary)` 始终正常发射以同步主界面已完成文件映射。
3. **`McpTranslationJob.cancel()`（`job_manager.py:1641-1645`）**：
   - 移除锁外冗余的 `if self._preview_pool is not None: self._preview_pool.shutdown(...)`，统一交由随后的 `_finalize_job_locked()` 持锁接管并关闭。

### 2.5 文案与 `allowed_origins` 微调（R3-5）

1. **`desktop_qt_ui/locales/zh_CN.json:1600`**：
   - 将 `"MCP Job Status Summary"` 中的 `"待渲染: {pending}"` 更新为 `"待翻页: {pending}"`。
2. **`manga_translator/mcp/server_app.py:71-77`**：
   - 在 `TransportSecuritySettings.allowed_origins` 中补齐 `"https://[::1]:*"`。

---

## 三、改动文件清单

| 文件路径 | 涉及修复项 |
| --- | --- |
| `manga_translator/mcp/job_manager.py` | R3-1（终态任务 `next_translation_batch` / `submit_translation_batch` 回退）、R3-2（分段末尾无字页/错误页封口与 `page.ctx = None`）、R3-4（`_handle_job_finished` 空闲判定与 `cancel()` 清理） |
| `manga_translator/mcp/batch_adapter.py` | R3-3（`retranslate_json` 补齐 `_prepare_loaded_regions` / `skip_font_scaling` / `pre_dict` 与 `load_text_parse_failures` 回写保护） |
| `manga_translator/mcp/server_app.py` | R3-5（`allowed_origins` 补齐 `https://[::1]:*`） |
| `desktop_qt_ui/services/mcp_service.py` | R3-4（存在新活跃任务时不覆盖 UI 进度卡片） |
| `desktop_qt_ui/locales/zh_CN.json` | R3-5（修正 `{pending}` 中文状态标签为 `"待翻页: {pending}"`） |
| `test/test_mcp_pipeline.py` | 新增 R3-1、R3-2、R3-3、R3-4 回归单测 |

---

## 四、分阶段实施步骤

1. **Phase 1 — 状态机与切批边界修复（R3-1、R3-2、R3-4）**
   - 在 `job_manager.py` 实现 `_build_done_payload_from_summary_locked`，并在 `next_translation_batch` 与 `submit_translation_batch` 中接入 `_completed_job_summaries` 回退。
   - 在 `_run_gpu_worker` 统一分段末尾与滑窗满时的 `_seal_building_batch_locked()` 调用，并补齐失败分支 `page.ctx = None`。
   - 收敛 `_handle_job_finished`、`cancel()` 与 `McpService._on_job_finished` 的异步收尾竞态。
2. **Phase 2 — `retranslate_json` 加固与文案/配置微调（R3-3、R3-5）**
   - 在 `batch_adapter.py` 补齐 `retranslate_json` 的 `_prepare_loaded_regions`、`skip_font_scaling`、`_apply_pre_dictionary_to_regions` 以及 `render_and_save_page` 的 `load_text_parse_failures` 回写拦截。
   - 更新 `zh_CN.json` 状态文案与 `server_app.py` 的 `https://[::1]:*`。
3. **Phase 3 — 自动化回归测试**
   - 在 `test/test_mcp_pipeline.py` 新增回归测试：
     - **R3-1a**：`pending_pages == 0`（全部跳过）时调用 `next_translation_batch(job_id)` 返回 `done=True` 及包含 `skipped_pages` 的 `summary`，不抛 `KeyError`；
     - **R3-1b**：`finalizing_render` 超时后等待后台渲染与 `_handle_job_finished` 裁剪完成（`job_id not in mgr._jobs`），再调用 `next_translation_batch(job_id)` 仍正常返回 `done=True`；
     - **R3-2**：构造 `[有字页 p1, 无字页 p2, 已跳过页 p3, 有字页 p4]` 且 `batch_size=3`，验证 `p1` 在分段末尾 `p2`（无字页）之后立即封口为独立批次，不与跳过页之后的 `p4` 合批；
     - **R3-3**：验证 `retranslate_json=True` 时 `pre_dict` 生效，且当 `load_text_parse_failures > 0` 时跳过 JSON 覆盖回写以保护原工程文件。
   - 运行 `pytest test/test_mcp_pipeline.py test/test_mcp_server.py` 确认全部通过。

---

## 五、验收标准

- `pending_pages == 0` 或任务完成后被 `_handle_job_finished` 从 `_jobs` 裁剪后，调用 `next_translation_batch(job_id)` 均稳定返回 `BatchPayload(done=True, ...)`，完全对齐 `skill_template.md` 契约。
- 跳过页分段末尾遇到无字页或预处理失败页时，前序累积批次严格在分段边界封口，绝不跨跳过页合批。
- `retranslate_json=True` 完整支持译前字典、缺失字号补齐与 JSON 解析失败写回熔断保护。
- `pytest test/test_mcp_pipeline.py test/test_mcp_server.py` 全部通过。
