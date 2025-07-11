//将非持久性数据置于系统剪贴板中。
Clipboard.SetDataObject("这条信息置于剪切板中,相当于 Ctrl+C");
//获取剪切板中文本格式的内容,相当于 Ctrl+V(不过如果剪切板中的内容不是文本格式就返回空字符串)
string message = Convert.ToString(Clipboard.GetDataObject().GetData(DataFormats.Text));
System.Windows.Forms 命名空间
Clipboard 类
Clipboard 成员
提供将数据置于系统剪贴板中以及从中检索数据的方法。无法继承此类。
方法:

| 名称                 | 说明                                                                                         |
| -------------------- | -------------------------------------------------------------------------------------------- |
| Clear                | 从剪贴板中移除所有数据。                                                                     |
| ContainsAudio        | 指示在剪贴板中是否存在 WaveAudio 格式的数据。                                                |
| ContainsData         | 指示剪贴板中是否存在指定格式的数据，或可转换成此格式的数据。                                 |
| ContainsFileDropList | 指示剪贴板中是否存在 FileDrop 格式或可转换成此格式的数据。                                   |
| ContainsImage        | 指示剪贴板中是否存在 Bitmap 格式或可转换成此格式的数据。                                     |
| ContainsText         | 已重载。 指示剪贴板中是否存在文本数据。                                                      |
| Equals               | 确定指定的 Object 是否等于当前的 Object。 （继承自 Object。）                                |
| Finalize             | 允许 Object 在“垃圾回收”回收 Object 之前尝试释放资源并执行其他清理操作。 （继承自 Object。） |
| GetAudioStream       | 检索剪贴板上的音频流。                                                                       |
| GetData              | 从剪贴板中检索指定格式的数据。                                                               |
| GetDataObject        | 检索当前位于系统剪贴板中的数据。                                                             |
| GetFileDropList      | 从剪贴板中检索文件名的集合。                                                                 |
| GetHashCode          | 用作特定类型的哈希函数。 （继承自 Object。）                                                 |
| GetImage             | 检索剪贴板上的图像。                                                                         |
| GetText              | 已重载。 从剪贴板中检索文本数据。                                                            |
| GetType              | 获取当前实例的 Type。 （继承自 Object。）                                                    |
| MemberwiseClone      | 创建当前 Object 的浅表副本。 （继承自 Object。）                                             |
| SetAudio             | 已重载。 将 WaveAudio 格式的数据添加到剪贴板中。                                             |
| SetData              | 将指定格式的数据添加到剪贴板中。                                                             |
| SetDataObject        | 已重载。 将数据置于系统剪贴板中。                                                            |
| SetFileDropList      | 将 FileDrop 格式的文件名集合添加到剪贴板中。                                                 |
| SetImage             | 将 Bitmap 格式的 Image 添加到剪贴板中。                                                      |
| SetText              | 已重载。 将文本数据添加到剪贴板中。                                                          |
| ToString             | 返回表示当前 Object 的 String。 （继承自 Object。）                                          |

C#定义了一个类 System.Windows.Forms.Clipboard 来简化剪切板操作，这个类有一个静态方法，主要有：

- Clear 清除剪切板中的所有数据；
- ContainsData，ContainsAudio，ContainsFlieDropList，ContainsText，ContainsImage，用于检查剪切板中是否存在相应的数据；
- GetAudioStream，GetDataObject，GetText，GetImage，GetFileDropList，用于取得数据；
- SetAudio，SetDataObject，SetText，SetImage，SetFileDropList，用于添加数据；

```csharp
using System;
using System.Text;
using System.Windows.Forms;
using System.Runtime.InteropServices;

namespace ChineseToEnglishPunctuation
{
    public partial class Form1 : Form
    {
        private IntPtr _nextClipboardViewerHandle;

        // Windows API constants
        private const int WM_DRAWCLIPBOARD = 0x308;
        private const int WM_CHANGECBCHAIN = 0x30D;

        // Windows API declarations
        [DllImport("user32.dll")]
        private static extern IntPtr SetClipboardViewer(IntPtr hWndNewViewer);

        [DllImport("user32.dll")]
        private static extern bool ChangeClipboardChain(IntPtr hWnd, IntPtr hWndNext);

        [DllImport("user32.dll")]
        private static extern int SendMessage(IntPtr hWnd, int msg, IntPtr wParam, IntPtr lParam);

        public Form1()
        {
            InitializeComponent();
        }

        protected override void OnLoad(EventArgs e)
        {
            base.OnLoad(e);
            _nextClipboardViewerHandle = SetClipboardViewer(Handle);
            if (_nextClipboardViewerHandle == IntPtr.Zero)
            {
                label1.Text = "Failed to register clipboard viewer.";
            }
        }

        protected override void WndProc(ref Message m)
        {
            try
            {
                switch (m.Msg)
                {
                    case WM_DRAWCLIPBOARD:
                        ProcessClipboardChange();
                        SendMessage(_nextClipboardViewerHandle, m.Msg, m.WParam, m.LParam);
                        break;

                    case WM_CHANGECBCHAIN:
                        UpdateClipboardChain(m);
                        break;

                    default:
                        base.WndProc(ref m);
                        break;
                }
            }
            catch (Exception ex)
            {
                label1.Text = $"Error: {ex.Message}";
            }
        }

        protected override void OnFormClosing(FormClosingEventArgs e)
        {
            base.OnFormClosing(e);
            ChangeClipboardChain(Handle, _nextClipboardViewerHandle);
        }

        private void ProcessClipboardChange()
        {
            if (!Clipboard.ContainsText())
            {
                return;
            }

            string text = Clipboard.GetText();
            if (string.IsNullOrEmpty( text))
            {
                return;
            }

            string convertedText = ConvertPunctuation(text);
            if (convertedText != text)
            {
                Clipboard.SetText(convertedText);
                label1.Text = "Punctuation converted successfully";
            }
        }

        private string ConvertPunctuation(string input)
        {
            StringBuilder result = new StringBuilder(input.Length);
            bool modified = false;

            foreach (char c in input)
            {
                char convertedChar = c;

                // Check if the character is in the full-width punctuation range (U+FF00–U+FFEF)
                if (c >= 0xFF00 && c <= 0xFFEF)
                {
                    // Map full-width punctuation to half-width (ASCII) equivalents
                    convertedChar = ConvertFullWidthToHalfWidth(c);
                    if (convertedChar != c)
                    {
                        modified = true;
                    }
                }

                result.Append(convertedChar);
            }

            return modified ? result.ToString() : input;
        }

        private char ConvertFullWidthToHalfWidth(char c)
        {
            // Map full-width punctuation to half-width (ASCII equivalents)
            switch (c)
            {
                // Full-width punctuation to half-width
                case '，': return ','; // U+FF0C -> U+002C
                case '。': return '.'; // U+FF0E -> U+002E
                case '；': return ';'; // U+FF1B -> U+003B
                case '：': return ':'; // U+FF1A -> U+003A
                case '？': return '?'; // U+FF1F -> U+003F
                case '！': return '!'; // U+FF01 -> U+0021
                case '（': return '('; // U+FF08 -> U+0028
                case '）': return ')'; // U+FF09 -> U+0029
                case '［': return '['; // U+FF3B -> U+005B
                case '］': return ']'; // U+FF3D -> U+005D
                case '｛': return '{'; // U+FF5B -> U+007B
                case '｝': return '}'; // U+FF5D -> U+007D
                case '＂': return '"'; // U+FF02 -> U+0022
                case '＇': return '\''; // U+FF07 -> U+0027
                case '＜': return '<'; // U+FF1C -> U+003C
                case '＞': return '>'; // U+FF1E -> U+003E
                default:
                    // For other full-width characters, attempt a general conversion
                    if (c >= 0xFF01 && c <= 0xFF5E)
                    {
                        // Full-width ASCII characters map to half-width by offset
                        return (char)(c - 0xFEE0);
                    }
                    return c; // No conversion needed
            }
        }

        private void UpdateClipboardChain(Message m)
        {
            if (m.WParam == _nextClipboardViewerHandle)
            {
                _nextClipboardViewerHandle = m.LParam;
            }
            else
            {
                SendMessage(_nextClipboardViewerHandle, m.Msg, m.WParam, m.LParam);
            }
        }
    }
}
```
