read me
import pyautogui
import time

print("自動移鼠功能已啟動。請將游標移至螢幕左上角以強制停止。")

try:
    while True:
        # 每 5 秒微幅移動滑鼠
        pyautogui.moveRel(1, 0, duration=0.25)
        pyautogui.moveRel(-1, 0, duration=0.25)
        time.sleep(5)
except KeyboardInterrupt:
    print("\n已停止自動移動。")
