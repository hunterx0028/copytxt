read me

import ctypes
import time

# Windows SetThreadExecutionState flags
ES_CONTINUOUS = 0x80000000
ES_SYSTEM_REQUIRED = 0x00000001
ES_DISPLAY_REQUIRED = 0x00000002

kernel32 = ctypes.windll.kernel32

try:
    # 告訴 Windows：
    # 1. 持續套用此狀態
    # 2. 系統不要因閒置進入睡眠
    # 3. 顯示器不要因閒置而關閉
    result = kernel32.SetThreadExecutionState(
        ES_CONTINUOUS | ES_SYSTEM_REQUIRED | ES_DISPLAY_REQUIRED
    )

    if result == 0:
        raise ctypes.WinError()

    print("已啟用：保持電腦喚醒及螢幕開啟")
    print("按 Ctrl+C 結束程式並恢復 Windows 原本的電源管理。")

    while True:
        time.sleep(60)

except KeyboardInterrupt:
    print("\n正在結束...")

finally:
    # 清除前面設定的要求，恢復 Windows 原本的電源管理
    kernel32.SetThreadExecutionState(ES_CONTINUOUS)
    print("已恢復 Windows 原本的電源管理設定。")
