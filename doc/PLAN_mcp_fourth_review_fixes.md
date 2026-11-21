# MCP 四次复审修复计划（Plan）

- **基准提交**：
  - `cffacd34afb4219fa1562c319cce9d51b54f8946`（`feat: Antigravity MCP`）
  - `3081dfcf29c1ec2eed7d578aa42089a8d3886c90`（`refactor: not only Antigravity`）
  - `91351eaf87236fe9147401386bf433e5f5286e0d`（`fix: review`）
  - `cca984076abb440f26473732d36b075b02d604db`（`fix: review again`）
  - `b48c1cdde68e9f09c6f5fc126c2547f16c3841f9`（`fix: review again again`）
- **改造目标**：修复第四次全量复审发现的 `retranslate_json` 缺失 `ctx.mask_raw` 导致真实掩码细化崩溃、`[BR]` 断句校验正则不一致与无重试上限、取消/在途提交/预览并发竞态与磁盘残留、R3-4 收尾竞态遗留（零待翻页 UI 进度不刷新与 `cancel_job` 双重空闲卸载）、`release_batch` 归档回退缺失、渲染终态判定不一致、GUI 停止任务同步阻塞主线程等问题，并补齐非 Mock 核心路径回归测试与跨客户端真机验收收口。
- **核心原则**：保持对外 MCP 工具契约与 0-API 硬隔离不变，仅做向后兼容的健壮性加固与行为一致性修正。

---

## 一、四次复审问题清单

| 编号 | 优先级 | 类别 | 涉及文件与位置 | 问题描述 |
| --- | --- | --- | --- | --- |
| R4-1 | 高 | 真实流水线崩溃 | `manga_translator/mcp/batch_adapter.py:341-352`、`864-874`；`manga_translator/mask_refinement/__init__.py:199` | `preprocess_page(..., retranslate_json=True)` 在原 JSON 无蒙版（`loaded_mask is None`）或为精修蒙版（`mask_is_refined=True`）时均未填充 `ctx.mask_raw`。前者进入 `run_page_inpaint`、后者因译文过滤触发 `pending_redo`（清空 `ctx.mask = None` 后重走 `_run_mask_refinement`）时，`dispatch_mask_refinement` 访问 `raw_mask.shape[0]` 必崩 `AttributeError`。此前单测因全部 Mock `_run_mask_refinement` 未暴露。此外 `retranslate_json` 未预热 `ctx.bubble_mask` 且未保留编辑器画笔/印章/贴片叠加层。 |
| R4-2 | 高 | 校验协议缺陷 | `manga_translator/mcp/batch_adapter.py:728-757`；`manga_translator/translators/common.py:2128` | ① `CommonTranslator._validate_br_markers`（`common.py:2128`）使用 `r'(\[BR\]|【BR】|<br>)'`（不匹配 `<br/>` / `<br />`），而 `batch_adapter.py:740` 构造 `errors` 时使用 `r"(\[BR\]|【BR】|<br\s*/?>)"`（匹配 `<br/>`），导致客户端提交 `<br/>` 时返回 `status="retry_needed", errors=[]`；② `[BR]` 校验未接入重试次数上限（仅 `enable_post_translation_check` 受 `retry_count < max_retries` 约束），若 OCR 误合并短气泡可能无限重试。 |
| R4-3 | 中 | 并发竞态与磁盘泄漏 | `manga_translator/mcp/job_manager.py:1175-1195`、`1279-1352`、`1431-1525`、`1567-1588`、`1637-1649` | ① `release_batch`（`1572` 行）未检查 `batch.submitting`，在提交校验/后处理期间调用会把批次重置为 `READY` 被另一 Worker 并发领走；② `cancel()` 不等待在途 `batch.submitting` 就执行 `_finalize_job_locked()` 销毁 `page.ctx`，且 `submit_translation_batch`（`1476` 行）持锁提交前未检查 `cancel_event / _finalized`，仍会提交已取消任务、写术语表 YAML 并在 `return_next=False` 时返回 `accepted`；③ `next_translation_batch`（`1175-1190`）与 `get_page_preview`（`1279-1352`）在终态后仍可能落盘预览图并泄漏 `mcp_preview/<job_id>` 目录，且 `get_page_preview` 未拦截已 `committed` / 渲染中的页面。 |
| R4-4 | 中 | 异步收尾竞态与 UI 同步 | `manga_translator/mcp/job_manager.py:408-410`、`1979-2010`、`2158-2165`；`desktop_qt_ui/services/mcp_service.py:373-382` | ① 全跳过任务（`pending_pages == 0`）在 `McpTranslationJob.__init__` 内即触发 `_finalize_job_locked()` → `McpService._on_job_finished`，此时 `create_translation_job` 仍持有 `self._creating_job = True` 且 `_active_job_id is None`，导致 `mcp_service.py:378` 判定失败、静默吞掉 `job_progress_changed`，GUI 进度卡片不刷新（同时 `377` 行无锁读取私有属性 `_active_job_id`）；② `cancel_job`（`2164` 行）漏加 R3-4 空闲判定且与 `_handle_job_finished` 重复调用 `_schedule_idle_cleanup_locked()`，当 `models_idle_ttl == 0` 时并发启动两个 `mcp-idle-unload` 线程；③ `PLAN_mcp_third_review_fixes.md` 要求的 R3-4 回归单测遗漏未写。 |
| R4-5 | 中 | 归档一致性与批次终态 | `manga_translator/mcp/job_manager.py:948-976`、`2082-2091`、`2130-2132` | ① `McpJobManager.release_batch` 未对齐 R3-1 的 `_completed_job_summaries` 回退，并发子代理在任务刚结束/取消后按 Skill 调用 `release_batch` 会抛未捕获 `KeyError`（`get_page_preview` 同理缺少明确终态提示）；② `_run_render_worker` 正常返回分支（`951` 行）整批失败仍硬设 `batch.status = "DONE"` 且未置 `page.ctx = None`，异常分支（`972` 行）部分成功却硬设 `batch.status = "FAILED"`，与 `888-892` / `1513-1517` 行规则不一致。 |
| R4-6 | 中 | UI 响应性 | `desktop_qt_ui/ui/main_page/pages/mcp_page.py:849-852`、`1112-1126`；`desktop_qt_ui/services/mcp_service.py:445-448` | 点击「停止当前任务」在 Qt 主线程同步调用 `cancel_active_job() -> job.cancel(join_timeout=5.0)`，串行 `join` GPU 线程和渲染线程最长阻塞主界面 10 秒。 |
| R4-7 | 低 | 边界行为与代码清理 | `manga_translator/mcp/job_manager.py:508`、`743-761`、`1607`、`1906-1996`；`manga_translator/mcp/protocol_models.py:200`；`manga_translator/mcp/batch_adapter.py:210-218`；`desktop_qt_ui/app_logic.py:49`；`desktop_qt_ui/locales/{zh_CN,en_US}.json` | ① 显式传入 `output_dir` 被 `cli.save_to_source_dir=True` 静默覆盖而返回值仍报 `output_dir`；② `create_translation_job` 展开压缩包后若后续步骤抛错未清理 `archive_to_temp_map`；③ 建任务未调用 `load_filter_list(force_reload=True)` 刷新外部修改的 `filter_list.json`；④ `failed_details[].page_index` 活任务为 `int`、状态/归档为 `str`；⑤ `job_manager.py:743-761` 空 `if ...: pass` 死代码、`app_logic.py:49` 未使用 `QMetaObject` 导入、`zh_CN.json:1585` / `en_US.json:1586` 内联预算提示 `1-50 MB` 与实际校验 `0.1-50 MB` 不符。 |

---

## 二、分项修复方案

### 2.1 修复 `retranslate_json` 掩码准备、气泡缓存与叠加层支持（R4-1）

1. **补全 `ctx.mask_raw`（`manga_translator/mcp/batch_adapter.py:341-366`）**：
   - 将 `translator._prepare_loaded_regions(loaded_regions, use_text_as_translation=False)` 提前到蒙版兜底构建之前，确保区域几何完整。
   - 若 `loaded_mask is not None`：按 `manga_translator.py:4128-4150` 规范化为 `(target_h, target_w)` 二维 `uint8` 数组；当 `mask_is_refined` 为 `True` 时，**同时**赋值 `ctx.mask = mask_arr` 与 `ctx.mask_raw = mask_arr`（`pending_redo` 清空 `ctx.mask` 后重跑 `dispatch_mask_refinement` 时，可基于保留的像素级文字蒙版按剩余 `text_regions` 精确裁剪）；当 `mask_is_refined` 为 `False` 时，赋值 `ctx.mask_raw = mask_arr`。
   - 若 `loaded_mask is None`：用 `loaded_regions` 的 `lines` 多边形填充生成兜底 `ctx.mask_raw`（与 `manga_translator.py:4114-4118` 一致；无区域时生成全零二维 `uint8` 掩码）。
2. **补齐气泡缓存预热**：当 `config.ocr.limit_mask_dilation_to_bubble_mask` 或 `config.ocr.use_model_bubble_repair_intersection` 或 `balloon_fill` 开启且 `hasattr(translator, "_prime_bubble_detection_cache")` 时，在 `preprocess_page(..., retranslate_json=True)` 中调用 `translator._prime_bubble_detection_cache(config, ctx)`。
3. **跨页并发安全的叠加层传递**：在 `preprocess_page(..., retranslate_json=True)` 紧接 `_load_text_and_regions_from_file` 后，把 `translator._loaded_render_overlays` 与 `translator._pending_paste_overlays` 摘取到 `ctx._mcp_loaded_render_overlays` / `ctx._mcp_pending_paste_overlays` 并清空 `translator` 实例暂存；在 `render_and_save_page`（`batch_adapter.py:928-944`，备份 `inpainted_snapshot` 之后、`_run_text_rendering` 之前）恢复并调用 `translator._compose_render_overlays_on_inpainted(ctx)` 后立即清理暂存，避免流水线多页串扰。

### 2.2 对齐 `[BR]` 断句正则并补齐重试上限（R4-2）

1. **统一断句标记正则（`manga_translator/translators/common.py:2128`）**：
   - 将 `_validate_br_markers` 第 2128 行 `re.search(r'(\[BR\]|【BR】|<br>)', ...)` 更新为 `re.search(r'(\[BR\]|【BR】|<br\s*/?>)', ...)`，与同方法第 2103 行单区域清理正则及 `batch_adapter.py:740` 完全一致。
2. **校验兜底与重试上限（`manga_translator/mcp/batch_adapter.py:724-753`）**：
   - 保持每次校验都调用 `helper._validate_br_markers(...)` 以确保单区域多余 `[BR]` 清理始终生效。
   - 计算 `max_br_retries = max(1, int(getattr(config.translator, "post_check_max_retry_attempts", 3) or 3))`：
     - 当 `not br_ok` 且 `retry_count < max_br_retries` 时，收集缺失 `[BR]` 的 `ValidationErrorItem`；若因任何原因 `errors` 为空，追加一条兜底 `ValidationErrorItem(id=None, reason="missing_br_marker", detail="Batch failed [BR] marker validation; ensure multi-line bubbles include [BR].")`，严禁返回 `status="retry_needed", errors=[]`。
     - 当 `not br_ok` 且 `retry_count >= max_br_retries` 时，记录 `logger.warning(...)` 并按容忍策略放行，终止无限重试循环。

### 2.3 收敛取消、在途提交、释放与按需看图并发竞态（R4-3）

1. **`release_batch` 保护在途提交（`manga_translator/mcp/job_manager.py:1567-1588`）**：
   - 在 `with self._cond:` 内检查 `if batch.submitting:`，返回 `{"released": False, "batch_id": batch_id, "status": batch.status, "message": f"Batch '{batch_id}' is currently being submitted and cannot be released."}`。
2. **`submit_translation_batch` 终态检查与 `cancel()` 等待在途提交（`job_manager.py:1431-1525`、`1637-1649`）**：
   - 在 `submit_translation_batch` 调用 `apply_batch_post_processing` 前（锁内）及写回提交状态前（`1476` 行锁内）均检查 `if self.cancel_event.is_set() or self._finalized:`，若已终态则直接返回 `SubmitBatchResponse(status="error", job_id=self.job_id, batch_id=batch_id, done=True, summary=self._build_completion_summary_locked(), message=f"Job '{self.job_id}' is already {self.status}.")`，避免修改 `page.committed` / `_render_queue` 或向磁盘写入已取消任务的术语。
   - 在 `McpTranslationJob.cancel()` 触发 `cancel_event.set()` 并 join GPU/Render 线程后，于 `_finalize_job_locked()` 前在 `self._cond` 上带超时等待 `not any(b.submitting for b in self._batches.values())`，确保无 HTTP 线程仍在读取 `page.ctx` 后再销毁内存缓冲。
3. **防止取消后残留预览文件与 `get_page_preview` 渲染期访问（`job_manager.py:1175-1195`、`1266-1369`）**：
   - `next_translation_batch` 在 `1189` 行检测到 `self.cancel_event.is_set() or self._finalized` 时，立即删除 `delivery_map` 中本次落盘的所有 `file_path` 并清理空 `self.preview_dir`（或再调用一次 `self._cleanup_temp_artifacts()`）。
   - `get_page_preview` 在取 `ctx` 前检查 `self.cancel_event.is_set() or self._finalized`、`page.failed` 以及 `page.committed or page.render_queued or page.rendered`（拒绝访问已进入渲染/释放流程的 `ctx`）；写入 `file_path` 后在锁内再次检查 `self.cancel_event.is_set() or self._finalized`，若已终态则删除刚写出的文件并清理 `preview_dir` 后抛 `RuntimeError`。

### 2.4 彻底闭合 R3-4 异步收尾与零待翻页 UI 同步（R4-4）

1. **延迟零待翻页任务的完成回调（`manga_translator/mcp/job_manager.py:408-410`、`525-568`、`1979-2010`）**：
   - 在 `McpTranslationJob._finalize_job_locked(self, *, defer_callback: bool = False)` 中支持延迟触发 `_on_finished`；`__init__` 对 `not self.pages` 传入 `defer_callback=True`，仅完成终态置位与临时目录清理。
   - 新增 `McpTranslationJob.dispatch_finished_callback()`，在 `create_translation_job` 持锁记录 `_latest_job_id` / `_completed_job_summaries` 并复位 `self._creating_job = False` **之后**调用，确保零待翻页任务进入 `McpService._on_job_finished` 时 `has_active_job()` 为 `False`，UI 进度卡片正常刷新。
2. **原子化 UI 进度更新判定（`job_manager.py` & `desktop_qt_ui/services/mcp_service.py:373-382`）**：
   - 在 `McpJobManager` 新增带锁方法 `should_update_ui_for_job(self, job_id: Optional[str]) -> bool`：在 `with self._lock:` 下若 `not self._creating_job and self._active_job_id is None` 返回 `True`，否则仅当 `job_id == self._active_job_id` 返回 `True`。
   - `McpService._on_job_finished` 改用 `if self.job_manager.should_update_ui_for_job(summary.get("job_id")):`，移除对私有字段 `_active_job_id` 的无锁读取。
3. **消除 `cancel_job` 与 `_handle_job_finished` 的双重空闲卸载（`job_manager.py:1748-1764`、`2158-2165`）**：
   - 在 `cancel_job` 与 `_handle_job_finished` 中以 `was_tracked = self._jobs.pop(job.job_id, None) is not None` 去重：仅当 `was_tracked`（或者 0 待翻页任务）且 `not self._creating_job and self._active_job_id is None` 时才调用 `_schedule_idle_cleanup_locked()`，杜绝 `cancel_job` 无视新任务调度卸载以及 `models_idle_ttl == 0` 时并发起两个 `mcp-idle-unload` 线程。

### 2.5 对齐 `release_batch` / `get_page_preview` 归档回退与统一渲染批次终态（R4-5）

1. **`McpJobManager.release_batch` 与 `get_page_preview` 归档查询（`job_manager.py:2082-2091`、`2130-2132`）**：
   - `release_batch` 在 `_jobs` 未命中时回退查询 `_completed_job_summaries`，命中则返回 `{"released": False, "batch_id": batch_id, "status": status_str, "message": f"Job '{target_id}' is already {status_str}."}` 而非抛 `KeyError`。
   - `get_page_preview` 在 `_jobs` 未命中但 `_completed_job_summaries` 命中时抛出明确 `RuntimeError(f"Job '{target_id}' is already {status_str} and its preview buffers have been released.")`。
2. **抽取 `_update_batch_terminal_status_locked` 并补齐 `page.ctx = None`（`job_manager.py:885-893`、`947-976`、`1512-1518`）**：
   - 统一在辅助方法中计算：`batch.status = "FAILED" if all(self.pages[i].failed for i in batch.page_indices) else "DONE"` 并调用 `self._delete_batch_preview_files(batch)`，4 处收敛点全部复用。
   - 在 `_run_render_worker` 正常渲染完成分支（`947` 行）补充 `page.ctx = None`，及时释放 `Context` 与 `TextBlock` 引用。

### 2.6 GUI「停止当前任务」异步化（R4-6）

1. **`McpService`（`desktop_qt_ui/services/mcp_service.py`）**：
   - 新增信号 `job_cancel_finished = pyqtSignal(bool, str)` 与方法 `cancel_active_job_async()`，在后台守护线程 `mcp-cancel-job` 中调用 `self.cancel_active_job()`，完成后在 `not self._shutdown_done` 时发射结果信号。
2. **`McpPagePanel`（`desktop_qt_ui/ui/main_page/pages/mcp_page.py`）**：
   - `_on_stop_job_clicked` 禁用 `self.stop_job_button` 并调用 `self.mcp_service.cancel_active_job_async()`。
   - 通过 `Qt.ConnectionType.QueuedConnection` 连接 `job_cancel_finished` 到 `_on_job_cancel_finished`（刷新状态并在报错时弹 Toast），并在 `McpPagePanel.shutdown()` 断连元组中同步补上 `job_cancel_finished`。

### 2.7 边界行为修正与代码清理（R4-7）

1. **显式 `output_dir` 覆盖 `save_to_source_dir`（`job_manager.py:1906-1916`、`1962`）**：若调用方显式传入非空 `output_dir`，强制 `save_to_source_dir = False`，确保实际落盘位置与 `CreateJobResponse.output_dir` 一致。
2. **`create_translation_job` 失败清理临时解压目录（`job_manager.py:1928-2010`）**：记录 `archive_to_temp_map` 及是否已移交 `McpTranslationJob`（`job_handed_off`）；若移交前抛错，在 `except` 中调用 `cleanup_archive_temp` 清理已解压的临时目录。
3. **刷新过滤列表缓存（`job_manager.py:1965` 或 `batch_adapter.py:210-218`）**：当 `translator.filter_text_enabled` 为 `True` 时调用 `load_filter_list(force_reload=True)`，与 `MangaTranslator.translate_batch` 保持一致。
4. **统一 `failed_details[].page_index` 类型（`protocol_models.py:200`、`job_manager.py:1607`）**：将 `JobStatusResponse.failed_details` 类型放宽为 `List[Dict[str, Any]]`，并在 `get_status_dict`（`1607` 行）保留 `int` 类型的 `p.page_index`，与 `_build_completion_summary_locked`（`508` 行）一致。
5. **死代码与文案微瑕清理**：
   - 删除 `job_manager.py:743-761` 中仅执行 `pass` 的空 `if` 判断。
   - 删除 `desktop_qt_ui/app_logic.py:49` 未使用的 `QMetaObject` 导入。
   - 将 `desktop_qt_ui/locales/zh_CN.json:1585` 与 `en_US.json:1586` 中的 `"MCP Inline Budget Hint"` 范围说明由 `1-50 MB` 更正为 `0.1-50 MB`。

---

## 三、改动文件清单

| 文件路径 | 涉及修复项 |
| --- | --- |
| `manga_translator/mcp/batch_adapter.py` | R4-1（`retranslate_json` 补全 `ctx.mask_raw`、气泡缓存与叠加层传递）、R4-2（`[BR]` 校验兜底错误与重试上限）、R4-7（刷新过滤列表缓存） |
| `manga_translator/translators/common.py` | R4-2（`_validate_br_markers` 第 2128 行支持 `<br\s*/?>`） |
| `manga_translator/mcp/job_manager.py` | R4-3（`release_batch` 拦截 `submitting`、`submit`/`cancel` 终态并发保护、预览磁盘残留清理与 `get_page_preview` 状态检查）、R4-4（延迟 0 待翻页回调、`should_update_ui_for_job`、`cancel_job` 去重空闲清理）、R4-5（`release_batch`/`get_page_preview` 归档回退、统一渲染批次终态与置空 `page.ctx`）、R4-7（`output_dir` 覆盖 `save_to_source_dir`、建任务失败清理压缩包、统一 `page_index` 为 `int`、删除空 `if`） |
| `manga_translator/mcp/protocol_models.py` | R4-7（`JobStatusResponse.failed_details` 放宽为 `List[Dict[str, Any]]`） |
| `desktop_qt_ui/services/mcp_service.py` | R4-4（使用 `should_update_ui_for_job`）、R4-6（新增 `job_cancel_finished` 信号与 `cancel_active_job_async`） |
| `desktop_qt_ui/ui/main_page/pages/mcp_page.py` | R4-6（停止按钮异步取消、信号连接与 `shutdown()` 断连） |
| `desktop_qt_ui/app_logic.py` | R4-7（移除未使用导入 `QMetaObject`） |
| `desktop_qt_ui/locales/zh_CN.json` | R4-7（修正 `"MCP Inline Budget Hint"` 范围为 `0.1-50 MB`） |
| `desktop_qt_ui/locales/en_US.json` | R4-7（修正 `"MCP Inline Budget Hint"` 范围为 `0.1-50 MB`） |
| `test/test_mcp_pipeline.py` | 新增 R4-1 ~ R4-5、R4-7 与此前遗漏的 R3-4 回归单测 |

---

## 四、分阶段实施步骤

1. **Phase 1 — 核心流水线与校验修复（R4-1、R4-2）**
   - 在 `batch_adapter.py` 补齐 `retranslate_json` 的 `ctx.mask_raw`（无蒙版多边形兜底 + `mask_is_refined=True` 保留 `mask_raw`）、气泡缓存预热及渲染叠加层上下文传递。
   - 在 `common.py` 对齐 `<br\s*/?>` 正则，并在 `batch_adapter.py` 补全 `[BR]` 校验兜底错误与最大重试次数保护。
2. **Phase 2 — 调度状态机并发与归档加固（R4-3、R4-4、R4-5）**
   - 在 `job_manager.py` 完成 `release_batch` `submitting` 拦截、`cancel()` 等待在途提交、`submit_translation_batch` 终态检查、取消后预览磁盘文件即时清理、`get_page_preview` 拒绝已提交/终态页面。
   - 实现零待翻页延迟回调、`should_update_ui_for_job`、`cancel_job` 空闲卸载去重、`release_batch`/`get_page_preview` 归档回退及统一 `_update_batch_terminal_status_locked` + `page.ctx = None`。
3. **Phase 3 — UI 异步取消与低优先级清理（R4-6、R4-7）**
   - 在 `mcp_service.py` 与 `mcp_page.py` 实现异步取消任务及信号断连。
   - 完成 `output_dir` 覆盖 `save_to_source_dir`、压缩包失败清理、`load_filter_list(force_reload=True)`、`failed_details.page_index` 统一为 `int`、清理空 `if` / `QMetaObject` / locales 文案。
4. **Phase 4 — 自动化回归测试**
   - 在 `test/test_mcp_pipeline.py` 新增以下测试用例：
     - **R4-1**：不 Mock `_run_mask_refinement`（直通真实 `dispatch_mask_refinement`），分别验证 `retranslate_json=True` 在「JSON 无蒙版」及「JSON 含精修蒙版 + 译文过滤触发 `pending_redo`」两种场景下均成功生成有效掩码并完成渲染保存；
     - **R4-2**：验证 `check_br_and_retry=True` 时提交 `<br/>` 正常通过且绝不返回 `errors=[]`；验证连续缺少 `[BR]` 达到重试上限后自动放行，不死循环；
     - **R4-3**：验证 `batch.submitting=True` 时 `release_batch` 被拒；验证提交中取消任务不置 `committed`、不写术语表；验证 `file_path` 落盘期间被取消不残留 `mcp_preview/<job_id>` 文件；
     - **R4-4（补齐 R3-4）**：验证延迟触发的旧任务 `_handle_job_finished` 与 `cancel_job` 不干扰新任务空闲定时器且不重复卸载；验证全跳过任务（`pending_pages == 0`）正常发射 `McpService.job_progress_changed`；
     - **R4-5 & R4-7**：验证已归档任务调用 `release_batch` 返回 `released=False`（不抛 `KeyError`）；验证整批渲染返回 `success=False` 时批次终态为 `FAILED` 且 `page.ctx is None`；验证显式传入 `output_dir` 时不被 `save_to_source_dir=True` 覆盖。
   - 运行 `.venv\Scripts\python.exe -m pytest test/test_mcp_pipeline.py test/test_mcp_server.py -v` 全部通过。
5. **Phase 5 — 跨客户端真机联调收口（沿袭前四份 Plan 遗留验收项）**
   - 抽测至少 2 个真实 MCP 客户端（Claude Code / Codex CLI / Cursor / OpenCode / Antigravity），端到端验证 `hq`、`text_on_demand`、`text_only`、`retranslate_json`、断点续传、UI 异步停止任务、显存卸载与关窗保护，并确认 Electron 宿主 `Origin` 头兼容性，全程 0 外部 LLM API 调用。

---

## 五、验收标准

- `retranslate_json=True` 在无蒙版 JSON 及精修蒙版触发 `pending_redo` 场景下走真实 `dispatch_mask_refinement` 零崩溃。
- `[BR]` 断句校验支持 `[BR]` / `【BR】` / `<br>` / `<br/>`，`retry_needed` 永不返回空 `errors`，且达到上限后安全收敛。
- 取消任务、并发释放、在途提交与按需看图之间无 `ctx` 释放后访问、无已取消任务写入、无临时预览目录残留。
- 全跳过任务正常刷新 GUI 进度卡片；`cancel_job` 不产生重复卸载线程也不影响紧随其后的新任务；点击 GUI「停止当前任务」不冻结主界面。
- `pytest test/test_mcp_pipeline.py test/test_mcp_server.py` 全部通过（含新增非 Mock 掩码细化与 R3-4/R4 回归用例），无新增警告。
