# Các phím tắt Github Copilot trong VS Code

| Phím tắt                 | Công dụng                             |
| ------------------------ | ------------------------------------- |
| `Ctrl + I`               | mở Inline Chat                        |
| `Ctrl + Alt + I`         | mở Chat view                          |
| `Ctrl + Shift + I`       | mở Copilot Edits view                 |
| `Ctrl + Shift + Alt + L` | mở Quick Chat                         |
| `Tab`                    | chấp nhận tất cả gợi ý                |
| `Ctrl + RightArrow`      | chấp nhận từng chữ gợi ý              |
| `Ctrl + LeftArrow`       | chấp nhận từng dòng gợi ý             |
| `Alt + ], Alt + [`       | hiển thị các tiếp theo và trước đó    |
| `Alt + \`                | mở gợi ý code                         |
| `Ctrl + /`               | thêm context                          |
| `Shift + Alt + T`        | chuyển văn bản đã chọn sang tiếng Anh |

## Chat variables

`#codebase`<br>
`#selection`<br>
`#new`<br>

## setting.json

```json
"github.copilot.nextEditSuggestions.enabled": true,
"github.copilot.chat.codesearch.enabled": true,
"github.copilot.enable": {
  "*": true,
  "plaintext": false,
  "markdown": false,
  "scminput": false
},
```

## Commands

- GitHub Copilot: Toggle (Enable/Disable) Completions
- Chat: Clear All Workspace Chats
- GitHub Copilot: Build Local Workspace Index
- GitHub Copilot: Build Remote Workspace Index
