# Customizations and mod suggestions
_Exported on 5/3/2026 at 23:07:18 GMT+7 from Cursor (3.2.16)_

---

**User**

@tiny11maker.ps1 xem xét các customize của bản này và đề xuất thêm các mod khác

---

**Cursor**

Đang đọc `tiny11maker.ps1` để nắm các tùy chỉnh hiện có và đề xuất bổ sung.



Dưới đây là **tóm tắt customize hiện có** trong `tiny11maker.ps1` và **hướng mở rộng** (mod thêm) — không sửa file trừ khi bạn muốn triển khai từng mục.

## Đang làm gì trong bản Reforged này

| Nhóm | Nội dung |
|------|----------|
| **Luồng** | GUI chọn ISO/ổ đĩa, chọn index edition; copy image, mount WIM, `ResetBase`, export nén, tạo ISO (`oscdimg`). |
| **AppX (Provisioned)** | Gỡ bộ Microsoft/Xbox/Bing/Copilot/Teams/Terminal/YourPhone/Clipchamp… (danh sách dài ~```266:318:tiny11maker.ps1```). |
| **Gói Windows (DISM Remove-Package)** | IE optional, LanguageFeatures (handwriting/OCR/speech/TTS theo `$languageCode`), Media Player, wallpaper extended, WordPad, Tablet PC Math, Steps Recorder (~```334:345:tiny11maker.ps1```). |
| **File hệ thống** | Xóa Edge / EdgeUpdate / EdgeCore / WebView2 runtime folder; xóa `OneDriveSetup.exe`. |
| **Registry (install.wim)** | Bypass TPM/CPU/RAM/SecureBoot/Storage; tắt “sponsored”/Content Delivery; `BypassNRO` + copy `autounattend.xml` vào Sysprep; tắt Reserved Storage; chặn BitLocker device encryption; tắt Chat/Taskbar Teams; tắt telemetry cơ bản; Copilot/Teams/Outlook policies; xóa vài scheduled task files; sửa quyền TaskCache. |
| **boot.wim (index 2)** | Lặp bypass phần cứng cho môi trường setup. |
| **Đầu ra** | `install.wim` → export sang `install.esd` (recovery), ISO gốc + `autounattend.xml` ở gốc ISO. |

---

## Lưu ý kỹ thuật (nên xử lý nếu bạn maintain script)

Các lệnh `reg load` đang dùng **chuỗi trong dấu nháy đơn** nên `$ScratchDisk` **không được thay thế** — đường dẫn có thể sai (ví dụ ~```383:387:tiny11maker.ps1``` và ~```564:566:tiny11maker.ps1```). Nên dùng chuỗi có mở rộng biến (ví dụ `"$ScratchDisk\scratchdir\..."`) hoặc nối chuỗi rõ ràng.

---

## Đề xuất mod bổ sung (chọn theo mục tiêu “gọn / riêng tư / ít nền”)

1. **Windows Capabilities (DISM `/Remove-Capability`)**  
   Có thể gỡ thêm nếu image có: `Print.Fax.Scan~~`, `WorkFolders-Client~~`, `Microsoft.Windows.PowerShell.ISE~~`, `OpenSSH*~~` (chỉ nếu bạn chắc không cần), v.v. — cần liệt kê từ `/Get-Capabilities` vì tên khác theo build.

2. **Delivery Optimization**  
   Policy tắt P2P / giảm nền: `HKLM\...\DeliveryOptimization` hoặc `Policies\Microsoft\Windows\DeliveryOptimization` (`DODownloadMode` = 0 hoặc tương đương theo doc Microsoft).

3. **Widgets / “tận cùng” thanh tác vụ**  
   Ví dụ `TaskbarDa` = 0 (Default User) để tắt widgets nếu vẫn còn trên build bạn target.

4. **Game DVR / Game Bar (policy)**  
   AppX Xbox đã bỏ nhiều; có thể thêm `GameDVR_Enabled` = 0 trong `HKLM\zNTUSER\...\GameBar` để tránh bật lại phần ghi màn hình.

5. **Activity History / quảng cáo thêm**  
   `PublishUserActivities` = 0, `EnableActivityFeed` = 0; có thể bổ sung khóa trong `Policies\Microsoft\Windows\System` tùy phiên bản.

6. **Windows Security / SmartScreen (tùy chọn, rủi ro)**  
   Một số bản “tiny” tắt SmartScreen/Defender — **giảm bảo mật rõ rệt**; chỉ nên đề cập nếu bạn chấp nhận rủi ro và kiểm tra build (24H2+ khác 22H2).

7. **Recall / AI (Windows 11 mới)**  
   Nếu ISO là 24H2+: policy tắt Recall / tính năng AI shell (tên khóa thay đổi theo bản; cần đối chiếu doc cho đúng build).

8. **Gói ngôn ngữ tùy chọn**  
   Hiện script gỡ Handwriting/OCR/Speech/TTS theo `$languageCode`; có thể mở rộng pattern cho `LanguageFeatures-Basic`, `Fonts-*`, v.v. **chỉ khi** bạn muốn ISO nhỏ hơn và chấp nhận mất tính năng nhập liệu/giọng nói.

9. **Tùy chọn người dùng (profile)**  
   `HKEY_USERS\.DEFAULT` đã có qua `zDEFAULT`/`zNTUSER`; có thể set thêm “Show file extensions”, “Open Explorer to This PC” — nhẹ, dễ chịu khi cài xong.

10. **Danh sách AppX theo build**  
    Trên bản mới thường xuất hiện thêm gói (ví dụ app “Cross Device”, biến thể Teams); nên sinh `$packagePrefixes` động từ một file JSON/txt kèm repo để không phải sửa script mỗi lần Microsoft thêm package.

Nếu bạn muốn bước tiếp theo là **tự động sửa lỗi `reg load` + thêm 2–3 mod cụ thể** (ví dụ Delivery Optimization + Widgets + Game DVR), trả lời **yes** và ưu tiên nhóm nào (chỉ privacy / chỉ dung lượng / cả hai).

