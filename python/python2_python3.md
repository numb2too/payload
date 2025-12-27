將 Exploit PoC 從 Python 2 移植到 Python 3 是滲透測試中非常常見的任務，尤其是針對舊漏洞（例如 Buffer Overflow 或是舊版 Web 應用）的腳本。

除了 `print` 需要括號之外，在 Exploit 開發的情境下，最關鍵的其實是 **「Bytes (字節) 與 String (字串) 的處理」**。

以下列出所有你需要注意的修改細節，按重要性排序：

### 1. 核心大魔王：Bytes vs String (最容易報錯的地方)

這是 Python 2 和 3 最大的差異。

- **Python 2:** `str` 和 `bytes` 幾乎通用，網路傳輸可以直接丟字串。
    
- **Python 3:** `str` 是 Unicode，`bytes` 是原始字節。**Socket 發送、Shellcode、Buffer 填充都必須是 `bytes` 類型。**
    

**常見修改場景：**

- **Socket 發送數據**
    
    - Py2: `s.send("GET / HTTP/1.1\r\n")`
        
    - Py3: `s.send(b"GET / HTTP/1.1\r\n")` (加上 `b` 前綴)
        
    - 或者: `s.send("string".encode())`
        
- **接收數據**
    
    - Py2: `data = s.recv(1024)` (收到的是 string，可以直接比對)
        
    - Py3: `data = s.recv(1024)` (收到的是 bytes，若要 print 出來給人看，最好 `data.decode()`)
        
- **Buffer Overflow Payload 串接**
    
    - Py2: `payload = "A" * 100 + "\x90" * 20`
        
    - Py3: `payload = b"A" * 100 + b"\x90" * 20` (必須確保所有部分都是 bytes)
        

### 2. 庫 (Library) 名稱變更

很多 PoC 會用到的網路庫在 Python 3 被重命名或整併了。

| **模組功能**          | **Python 2 (舊)**                    | **Python 3 (新)**                                 |
| ----------------- | ----------------------------------- | ------------------------------------------------ |
| **網頁請求**          | `import urllib2`<br>`import urllib` | `import urllib.request`<br>`import urllib.parse` |
| **URL 編碼**        | `urllib.quote()`                    | `urllib.parse.quote()`                           |
| **HTTP Client**   | `import httplib`                    | `import http.client`                             |
| **簡易伺服器**         | `python -m SimpleHTTPServer`        | `python -m http.server`                          |
| **Socket Server** | `import SocketServer`               | `import socketserver` (全小寫)                      |

### 3. 異常處理 (Exception Handling) 語法

舊的寫法在 Python 3 會直接 Syntax Error。

- Py2: `except Exception, e:`
    
- Py3: `except Exception as e:`
    

### 4. 數學運算：除法 (重要！)

如果 Exploit 涉及到計算 Offset 或 Padding 長度，這點非常危險。

- **Python 2:** `5 / 2` 結果是 `2` (整數除法)。
    
- **Python 3:** `5 / 2` 結果是 `2.5` (浮點數除法)。
    
- **修正:** 如果你需要整數結果（例如計算 Payload 長度），請在 Python 3 使用 `//` (Floor Division)。
    
    - `offset = total_len // 2`
        

### 5. 輸入函數

如果 PoC 需要使用者交互：

- Py2: `raw_input("Command: ")`
    
- Py3: `input("Command: ")`
    
    - _注意：Py2 的 `input()` 會執行代碼，Py3 的 `input()` 行為等於 Py2 的 `raw_input()`。_
        

### 6. 迭代器與列表 (Iterators vs Lists)

這通常不影響 Exploit 功能，但會影響記憶體或報錯。

- **Range:**
    
    - Py2: `xrange()` (返回迭代器) / `range()` (返回列表)
        
    - Py3: `range()` (返回迭代器)
        
    - _修改：將 `xrange` 全部改為 `range` 即可。_
        
- **字典遍歷:**
    
    - Py2: `d.iteritems()`
        
    - Py3: `d.items()`
        

### 7. struct 模組 (Binary Data Packing)

Exploit 很常使用 `struct.pack` 來將記憶體地址轉為 Little Endian 格式。

- `struct.pack('<I', 0xdeadbeef)`
    
- 在 Python 3 中，這個函數返回的是 **bytes**。如果你試圖把它和普通字串 (`str`) 相加，會報錯 `TypeError: can only concatenate str (not "bytes") to str`。
    
- **解法：** 確保與它相加的變數（如 padding `A` * 100）也是 bytes 類型 (`b"A"*100`)。
    

---

### 快速轉換工具 tip

你可以使用 Python 內建的 `2to3` 工具來自動處理大部分語法問題（如 print, except, xrange）：

Bash

```
2to3 -w exploit.py
```

- `-w` 代表直接寫入檔案修改。
    
- **警告：** `2to3` **無法** 自動修復 Bytes/String 的邏輯問題，這部分你必須手動檢查 `socket.send` 和 `payload` 組合的地方。
    

### 總結 Checklist

1. [ ] `print` 加上 `()`。
    
2. [ ] `socket.send()` 內的內容加上 `b""` 或 `.encode()`。
    
3. [ ] 檢查所有 Payload 拼接，確保是 `bytes + bytes`。
    
4. [ ] `urllib2` 改為 `urllib.request`。
    
5. [ ] `except Exception, e` 改為 `as e`。
    
6. [ ] `raw_input` 改為 `input`。
    
7. [ ] 計算長度的除法 `/` 確認是否要改成 `//`。
    
