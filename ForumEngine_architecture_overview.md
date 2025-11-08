# Tổng quan Kiến trúc ForumEngine

## Giới thiệu

ForumEngine là một thành phần quan trọng trong hệ thống BettaFish, hoạt động như một "diễn đàn ảo" thông minh để giám sát, điều phối và tổng hợp hoạt động của ba động cơ chính (InsightEngine, MediaEngine, QueryEngine). ForumEngine không chỉ là một công cụ ghi log đơn thuần mà còn là một hệ thống điều phối thông minh, tạo ra không gian giao tiếp có cấu trúc giữa các AI Agent.

## Mục đích Thiết kế

ForumEngine được thiết kế để giải quyết các vấn đề sau:

1. **Giám sát Thời gian thực**: Theo dõi hoạt động của tất cả các động cơ trong hệ thống
2. **Tổng hợp Thông tin**: Trích xuất và định dạng các kết quả quan trọng từ log
3. **Điều phối Giao tiếp**: Tạo ra không gian "thảo luận" giữa các AI Agent
4. **Quản lý Phiên**: Kiểm soát vòng đời của các phiên nghiên cứu
5. **Cung cấp Bối cảnh**: Giúp các động cơ hiểu được hoạt động của nhau

## Kiến trúc Chi tiết

### Cấu trúc Thư mục

```
ForumEngine/
├── __init__.py           # Khởi tạo module
├── monitor.py           # Core monitoring logic
└── llm_host.py          # Host speech generation
```

### Các Thành phần Chính

#### 1. LogMonitor (monitor.py)

Là thành phần trung tâm của ForumEngine, chịu trách nhiệm:

- **Giám sát File**: Theo dõi 3 file log chính:
  - `logs/insight.log` - Log từ InsightEngine
  - `logs/media.log` - Log từ MediaEngine  
  - `logs/query.log` - Log từ QueryEngine

- **Trích xuất Nội dung**: Nhận diện và trích xuất các nội dung quan trọng:
  - FirstSummaryNode outputs
  - ReflectionSummaryNode outputs
  - JSON outputs từ các node

- **Quản lý Phiên**: 
  - Bắt đầu phiên khi phát hiện FirstSummaryNode
  - Kết thúc phiên khi không có hoạt động mới
  - Tự động làm mới sau mỗi phiên

- **Điều phối Host**: 
  - Thu thập các phát biểu từ agent
  - Kích hoạt tạo host speech sau mỗi 5 phát biểu
  - Tích hợp host speech vào diễn đàn

#### 2. Host Speech Generation (llm_host.py)

Cung cấp khả năng tạo ra các phát biểu điều phối thông minh:

- **Tổng hợp Thông tin**: Phân tích các phát biểu gần đây từ các agent
- **Tạo Điều phối**: Sinh ra các bình luận tổng hợp, điều phối
- **Cung cấp Bối cảnh**: Giúp các agent hiểu được tiến trình chung

## Luồng hoạt động Chi tiết

### 1. Khởi tạo Hệ thống

```python
# Khởi tạo LogMonitor
monitor = LogMonitor(log_dir="logs")

# Khởi tạo file forum.log
monitor.clear_forum_log()

# Bắt đầu giám sát
monitor.start_monitoring()
```

### 2. Vòng lặp Giám sát Chính

```python
def monitor_logs(self):
    while self.is_monitoring:
        # Kiểm tra thay đổi file
        for app_name, log_file in self.monitored_logs.items():
            new_lines = self.read_new_lines(log_file, app_name)
            
            # Xử lý các dòng mới
            if new_lines:
                captured_contents = self.process_lines_for_json(new_lines, app_name)
                
                # Ghi vào forum.log
                for content in captured_contents:
                    self.write_to_forum_log(content, app_name.upper())
```

### 3. Xử lý Nội dung Thông minh

#### Nhận diện Target Node

ForumEngine chỉ xử lý các log từ các node quan trọng:

```python
target_node_patterns = [
    'FirstSummaryNode',           # Node tóm tắt ban đầu
    'ReflectionSummaryNode',      # Node tóm tắt phản ánh
    'InsightEngine.nodes.summary_node',    # Module path
    'MediaEngine.nodes.summary_node',
    'QueryEngine.nodes.summary_node',
    'nodes.summary_node',         # Partial match
    '正在生成首次段落总结',         # Chinese identifiers
    '正在生成反思总结',
]
```

#### Xử lý JSON đa dòng

ForumEngine có khả năng xử lý JSON output phức tạp:

```python
def extract_json_content(self, json_lines: List[str]) -> Optional[str]:
    # Tìm vị trí bắt đầu JSON
    # Ghép các dòng JSON bị chia cắt
    # Sửa lỗi JSON nếu cần
    # Trích xuất nội dung chính
```

#### Lọc ERROR Block

ForumEngine thông minh lọc các ERROR block:

```python
def process_lines_for_json(self, lines: List[str], app_name: str):
    for line in lines:
        log_level = self.get_log_level(line)
        
        if log_level == 'ERROR':
            # Bắt đầu ERROR block
            self.in_error_block[app_name] = True
            continue
        elif log_level == 'INFO':
            # Kết thúc ERROR block
            self.in_error_block[app_name] = False
```

### 4. Quản lý Phiên Thông minh

#### Bắt đầu Phiên

```python
# Phát hiện FirstSummaryNode
if 'FirstSummaryNode' in line or '正在生成首次段落总结' in line:
    logger.info(f"Phát hiện nội dung đầu tiên từ {app_name}")
    self.is_searching = True
    self.clear_forum_log()  # Làm mới diễn đàn
```

#### Kết thúc Phiên

```python
# Kiểm tra điều kiện kết thúc
if not any_growth and not captured_any:
    self.search_inactive_count += 1
    if self.search_inactive_count >= 900:  # 15 phút không hoạt động
        self.is_searching = False
        self.write_to_forum_log("=== ForumEngine 论坛结束 ===", "SYSTEM")
```

### 5. Điều phối Host Speech

#### Thu thập Phát biểu Agent

```python
# Thêm vào buffer
self.agent_speeches_buffer.append(log_line)

# Kiểm tra ngưỡng
if len(self.agent_speeches_buffer) >= self.host_speech_threshold:
    self._trigger_host_speech()
```

#### Tạo Host Speech

```python
def _trigger_host_speech(self):
    # Lấy 5 phát biểu gần nhất
    recent_speeches = self.agent_speeches_buffer[:5]
    
    # Tạo host speech
    host_speech = generate_host_speech(recent_speeches)
    
    # Ghi vào diễn đàn
    self.write_to_forum_log(host_speech, "HOST")
    
    # Xóa đã xử lý
    self.agent_speeches_buffer = self.agent_speeches_buffer[5:]
```

## Định dạng Log và Giao tiếp

### Định dạng Log File

```
[HH:MM:SS] [SOURCE] CONTENT

Ví dụ:
[14:30:15] [INSIGHT] Dựa trên phân tích dữ liệu từ微博...
[14:30:45] [MEDIA] Nội dung video cho thấy xu hướng...
[14:31:20] [HOST] Các engine đã cung cấp góc nhìn đa chiều về...
```

### Giao tiếp với Frontend

ForumEngine cung cấp API để frontend truy cập:

```python
@app.route('/api/forum/log')
def get_forum_log():
    # Đọc forum.log
    # Phân tích các dòng
    # Trả về JSON format
    return jsonify({
        'log_lines': lines,
        'parsed_messages': parsed_messages,
        'total_lines': len(lines)
    })
```

## Tích hợp với Hệ thống

### 1. Tích hợp với Flask App

```python
# app.py
def start_forum_engine():
    from ForumEngine.monitor import start_forum_monitoring
    start_forum_monitoring()

def stop_forum_engine():
    from ForumEngine.monitor import stop_forum_monitoring
    stop_forum_monitoring()
```

### 2. Giám sát Thời gian thực

```python
# Thread giám sát riêng
forum_monitor_thread = threading.Thread(target=monitor_forum_log, daemon=True)
forum_monitor_thread.start()

def monitor_forum_log():
    while True:
        # Đọc thay đổi forum.log
        # Gửi qua WebSocket đến frontend
        socketio.emit('forum_message', parsed_message)
        time.sleep(1)
```

### 3. Tích hợp với ReportEngine

ReportEngine sử dụng forum.log để tạo báo cáo tổng hợp:

```python
def load_input_files(self, file_paths: Dict[str, str]):
    # Đọc báo cáo từ 3 engine
    # Đọc forum.log cho bối cảnh
    with open(file_paths['forum'], 'r', encoding='utf-8') as f:
        content['forum_logs'] = f.read()
```

## Các Tính năng Nâng cao

### 1. Xử lý JSON Thông minh

- **Phát hiện JSON đa dòng**: Nhận diện JSON bị chia cắt qua nhiều dòng log
- **Sửa lỗi JSON**: Tự động sửa các lỗi JSON phổ biến
- **Trích xuất nội dung**: Lấy ra nội dung chính từ JSON structure

### 2. Quản lý Bộ nhớ

- **Cleanup buffer**: Tự động dọn dẹp buffer để tránh memory leak
- **Line hash tracking**: Sử dụng hash để tránh xử lý trùng lặp
- **Position tracking**: Theo dõi vị trí đọc file hiệu quả

### 3. Xử lý Đa ngôn ngữ

- **Hỗ trợ Trung Quốc**: Nhận diện các log tiếng Trung
- **Unicode handling**: Xử lý đúng encoding UTF-8
- **Multi-platform**: Hoạt động trên cả Windows và Unix

### 4. Error Handling

- **Graceful degradation**: Tiếp tục hoạt động khi có lỗi
- **Fallback mechanisms**: Có các phương án dự phòng
- **Comprehensive logging**: Ghi log chi tiết cho debugging

## Cấu hình và Tùy chỉnh

### Các Tham số Cấu hình

```python
class LogMonitor:
    def __init__(self, log_dir: str = "logs"):
        # Thời gian chờ không hoạt động (15 phút)
        self.search_inactive_count = 0
        
        # Ngưỡng kích hoạt host speech (5 phát biểu)
        self.host_speech_threshold = 5
        
        # File log output
        self.forum_log_file = self.log_dir / "forum.log"
```

### Tùy chỉnh Node Patterns

```python
target_node_patterns = [
    # Thêm các pattern mới nếu cần
    'CustomNode',
    'CustomEngine.nodes.custom_node',
]
```

## Hiệu suất và Tối ưu

### 1. I/O Efficiency

- **Non-blocking reads**: Sử dụng non-blocking I/O để tránh lag
- **Buffer management**: Quản lý buffer hiệu quả
- **File position tracking**: Theo dõi vị trí file thay vì đọc lại

### 2. Memory Management

- **Line limit**: Giới hạn số dòng trong buffer
- **Periodic cleanup**: Dọn dẹp định kỳ
- **Efficient data structures**: Sử dụng cấu trúc dữ liệu tối ưu

### 3. CPU Optimization

- **Regex optimization**: Tối ưu hóa các biểu thức chính quy
- **Minimal processing**: Chỉ xử lý các dòng cần thiết
- **Async operations**: Sử dụng threading cho các tác vụ I/O

## Troubleshooting và Debugging

### Các Vấn đề Thường gặp

1. **File Permission**: Đảm bảo có quyền đọc/ghi file log
2. **Encoding Issues**: Kiểm tra UTF-8 encoding
3. **Memory Usage**: Monitor buffer sizes
4. **Performance**: Kiểm tra I/O bottleneck

### Debug Tools

```python
# Test log writing
@app.route('/api/test_log/<app_name>')
def test_log(app_name):
    test_msg = f"[{datetime.now().strftime('%H:%M:%S')}] Test message"
    write_log_to_file(app_name, test_msg)
    return jsonify({'success': True})

# Manual forum control
@app.route('/api/forum/start')
def start_forum_monitoring_api():
    start_forum_monitoring()
    return jsonify({'success': True})
```

## Kết luận

ForumEngine là một thành phần kiến trúc thông minh, đóng vai trò như "bộ não" điều phối của hệ thống BettaFish. Với khả năng:

- **Giám sát thông minh** các hoạt động của hệ thống
- **Tổng hợp hiệu quả** thông tin từ nhiều nguồn
- **Điều phối linh hoạt** giao tiếp giữa các AI Agent
- **Quản lý tự động** vòng đời các phiên nghiên cứu

ForumEngine không chỉ giúp hệ thống hoạt động hiệu quả hơn mà còn tạo ra một không gian làm việc hợp tác thông minh giữa các AI Agent, giúp BettaFish tạo ra các báo cáo phân tích dư luận xã hội toàn diện và sâu sắc.