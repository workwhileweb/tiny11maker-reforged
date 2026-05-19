# Optimizer script performance improvement
_Exported on 5/8/2026 at 09:20:44 GMT+7 from Cursor (3.3.22)_

---

**User**

@tiny11maker.ps1  đọc kỹ, phân tích các optimizer script này thực hiện, đề xuất các cải tiến và more mod? code hiện tại chiếm dụng cpu rất lớn, làm các app khác gần như treo cứng, có thể cải tiến để chạy ngầm hoặc giảm lượng cpu không?


---

**Cursor**

Đang đọc `tiny11maker.ps1` và các script liên quan để phân tích và đề xuất cải tiến về CPU.



Đã nắm luồng chính. Phát hiện thêm: module gọi `New-ModuleManifest` mỗi lần import và vòng lặp GUI `DoEvents`. Đang kiểm tra scope biến và triển khai một số cải tiến giảm tải CPU.



Phân tích hoàn chỉnh và áp dụng các cải tiến an toàn: giảm tải khi chờ GUI, ưu tiên tiến trình thấp khi build, tránh ghi manifest mỗi lần import, và cache `Add-Type` cho `Enable-Privilege`.



Chỉ áp dụng BelowNormal khi bật `-LowPriority`. `-FastCompression` chỉ đổi tham số nén.



Dưới đây là phân tích ngắn gọn, hướng “mod” thêm, và những gì đã chỉnh trong repo.

---

## `tiny11maker.ps1` làm gì (pipeline “optimizer”)

1. **GUI** (`tiny11gui.psm1`): chọn mount ISO hoặc ổ đã gắn, chọn edition/index.  
2. **DISM + copy**: export `install.esd` → `install.wim` (nếu cần), `Copy-Item` cả cây `tiny11`, mount WIM vào `scratchdir`.  
3. **Gỡ Appx provisioned**: danh sách prefix (Xbox, Copilot, Clipchamp, …) → nhiều lần `DISM /Remove-ProvisionedAppxPackage`.  
4. **Gói Windows tùy chọn**: IE optional, Language Features, Media Player, WordPad, v.v. → `DISM /Remove-Package`.  
5. **File**: xóa Edge / EdgeUpdate / WebView2, OneDrive setup.  
6. **Registry (offline)**: `reg load` các hive → bypass TPM/CPU/RAM, tắt telemetry/sponsored apps, OOBE local account, Reserved Storage, BitLocker device encryption, Copilot, Teams, scheduled task files, v.v.  
7. **DISM** `Cleanup-Image /StartComponentCleanup /ResetBase` → unmount/commit → export WIM (nén).  
8. **boot.wim** index 2: tương tự bypass requirement, unmount.  
9. **Export** `install.esd` (`/Compress:recovery`), copy `autounattend.xml`, **oscdimg** tạo ISO.

Phần “nặng” thực sự là **DISM (đặc biệt nén max + cleanup + export ESD)** và **copy hàng chục GB** — đó là I/O + CPU, không phải vòng lặp PowerShell thuần.

---

## Vì sao máy “gần treo” / CPU cao

| Nguồn | Ghi chú |
|--------|--------|
| `DISM /Compress:max` | Nén mạnh = CPU cao, lâu. |
| `Cleanup-Image /ResetBase` | Dọn component store, tốn CPU/I/O. |
| Export sang `.esd` `/Compress:recovery` | Vẫn nặng. |
| `Copy-Item -Recurse` toàn bộ ISO | Đĩa bão hòa → hệ thống chậm. |
| Tiến trình **Normal** priority | Cạnh tranh với app khác. |

**Chạy “ngầm”** (ẩn cửa sổ) chỉ giúp không nhìn thấy console; **không** giảm tải CPU/DISM. Để nhẹ hơn cho app khác cần **hạ priority** và/hoặc **nén nhanh hơn** / bớt bước nặng.

---

## Cải tiến đã áp dụng trong code

1. **`tiny11maker.ps1`**  
   - `-LowPriority`: sau khi đóng GUI, đặt `PriorityClass = BelowNormal` cho tiến trình PowerShell hiện tại → OS ưu tiên app khác hơn (build chậm hơn một chút nhưng máy đỡ “đơ”).  
   - `-FastCompression`: các bước `DISM /Export-Image` dùng `/Compress:fast` thay cho `max` → **giảm CPU và thời gian**, `install.wim` **lớn hơn**; bước `.esd` recovery giữ nguyên.

2. **`lib/tiny11gui.psm1`**  
   - `Update-EventLoop`: `Start-Sleep` **33ms → 100ms** để bớt gọi `DoEvents` (phần chờ form ít tốn CPU hơn).  
   - `Set-DrivesList` / `Set-EditionsList`: gán **`$script:LIST_*`** để list thực sự vào scope module (tránh lỗi shadowing).  
   - `New-ModuleManifest` chỉ chạy **khi chưa có file `.psd1`** (trước đây mỗi lần `Import-Module` đều ghi manifest).

3. **`lib/tiny11utils.psm1`**  
   - Cùng quy tắc `New-ModuleManifest`.  
   - `Enable-Privilege`: **`Add-Type` một lần** (cache type `AdjPriv`), gọi tĩnh `[AdjPriv]::EnablePrivilege([long]$handle, ...)` thay vì compile mỗi lần.

**Cách chạy ví dụ** (sau khi chỉnh quyền admin như cũ):

```powershell
.\tiny11maker.ps1 -ScratchDisk "D:\work" -LowPriority -FastCompression
```

(Nếu vẫn chỉ truyền thư mục như batch cũ, tham số đầu vẫn là `$ScratchDisk`.)

---

## Đề xuất “more mod” / tinh chỉnh thêm (chưa code)

- **Tùy chọn bỏ `/ResetBase`** hoặc chỉ `StartComponentCleanup` không reset base → build nhanh hơn, image có thể lớn hơn.  
- **Robocopy** thay `Copy-Item` với `/MT:2` hoặc `/IPG` để giảm bão đĩa (chậm hơn nhưng mượt hơn cho hệ thống).  
- **CLI không GUI**: `-IsoPath`, `-Drive`, `-Index`, `-NoGui` để chạy từ task / scheduler minimized.  
- **Sửa filter Appx** (đoạn `packagesToRemove` với `-contains` và `Where-Object` lồng nhau trông không đúng logic prefix; có thể là bug hoặc may khớp tình cờ).  
- Mod registry/app list theo “preset”: Gaming / Office / Minimal.  
- Trùng `Microsoft.WindowsTerminal` trong `$packagePrefixes` (hai lần) — có thể gọn lại.

---

## Kết luận

- CPU cao chủ yếu do **DISM + copy + nén**, không phải do GUI sau khi đã chỉnh sleep.  
- **`-LowPriority`** và **`-FastCompression`** là cách thực tế để hệ thống **đỡ nghẽn** và **bớt CPU** khi build, đổi lại tốc độ/ dung lượng WIM.  
- Chạy ngầm chỉ ẩn UI; muốn nhẹ cho máy thì ưu tiên **priority + nén + (tùy chọn) bớt ResetBase / robocopy throttle**.

