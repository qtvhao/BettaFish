# Tổng quan Kiến trúc InsightEngine

## Giới thiệu

InsightEngine là một trong ba động cơ nghiên cứu chính của hệ thống BettaFish, chuyên biệt trong việc phân tích dư luận xã hội dựa trên cơ sở dữ liệu địa phương. InsightEngine sử dụng các công cụ tìm kiếm chuyên dụng để trích xuất thông tin từ cơ sở dữ liệu MediaCrawler, kết hợp với khả năng phân tích cảm xúc đa ngôn ngữ để tạo ra các báo cáo phân tích sâu và toàn diện.

## Mục đích Thiết kế

InsightEngine được thiết kế để:

1. **Phân tích Dữ liệu Địa phương**: Truy cập và phân tích dữ liệu từ cơ sở dữ liệu MediaCrawler
2. **Nghiên cứu Sâu**: Thực hiện các vòng lặp nghiên cứu và phản ánh để đạt hiểu biết sâu sắc
3. **Phân tích Cảm xúc**: Tích hợp phân tích cảm xúc đa ngôn ngữ (22 ngôn ngữ)
4. **Tối ưu hóa Từ khóa**: Sử dụng middleware tối ưu hóa từ khóa để cải thiện kết quả tìm kiếm
5. **Tạo Báo cáo Chuyên nghiệp**: Sinh ra các báo cáo phân tích dư luận chi tiết và có cấu trúc

## Kiến trúc Chi tiết

### Cấu trúc Thư mục

```
InsightEngine/
├── __init__.py                    # Khởi tạo module
├── agent.py                       # DeepSearchAgent chính
├── llms/                          # LLM integration
│   ├── __init__.py
│   └── base.py                    # LLM client base
├── nodes/                         # Processing nodes
│   ├── __init__.py
│   ├── base_node.py               # Base node class
│   ├── report_structure_node.py   # Report structure generation
│   ├── search_node.py             # Search query generation
│   ├── summary_node.py            # Content summarization
│   └── formatting_node.py         # Report formatting
├── state/                         # State management
│   ├── __init__.py
│   └── state.py                   # State data structures
├── tools/                         # External tools
│   ├── __init__.py
│   ├── search.py                  # MediaCrawlerDB tools
│   ├── keyword_optimizer.py       # Keyword optimization
│   └── sentiment_analyzer.py      # Sentiment analysis
├── prompts/                       # System prompts
│   ├── __init__.py
│   └── prompts.py                 # All prompt definitions
└── utils/                         # Utilities
    ├── __init__.py
    ├── db.py                      # Database connection
    └── text_processing.py         # Text processing utilities
```

### Các Thành phần Chính

#### 1. DeepSearchAgent (agent.py)

Là thành phần trung tâm điều phối toàn bộ quá trình nghiên cứu:

```python
class DeepSearchAgent:
    def __init__(self, config: Optional[Settings] = None):
        self.config = config or settings
        self.llm_client = self._initialize_llm()
        self.search_agency = MediaCrawlerDB()
        self.sentiment_analyzer = multilingual_sentiment_analyzer
        self._initialize_nodes()
        self.state = State()
```

**Các phương thức chính:**
- `research(query, save_report=True)` - Thực hiện nghiên cứu hoàn chỉnh
- `execute_search_tool(tool_name, query, **kwargs)` - Thực thi công cụ tìm kiếm
- `analyze_sentiment_only(texts)` - Phân tích cảm xúc độc lập

#### 2. Processing Nodes

##### BaseNode (base_node.py)

Lớp cơ sở cho tất cả các processing nodes:

```python
class BaseNode(ABC):
    def __init__(self, llm_client: LLMClient, node_name: str = ""):
        self.llm_client = llm_client
        self.node_name = node_name or self.__class__.__name__
    
    @abstractmethod
    def run(self, input_data: Any, **kwargs) -> Any:
        pass
```

##### ReportStructureNode (report_structure_node.py)

Tạo cấu trúc báo cáo từ truy vấn người dùng:

```python
class ReportStructureNode(StateMutationNode):
    def run(self, input_data: Any = None, **kwargs) -> List[Dict[str, str]]:
        # Gọi LLM để tạo cấu trúc báo cáo
        response = self.llm_client.stream_invoke_to_string(
            SYSTEM_PROMPT_REPORT_STRUCTURE, 
            self.query
        )
        return self.process_output(response)
```

##### SearchNode (search_node.py)

Tạo các truy vấn tìm kiếm thông minh:

- **FirstSearchNode**: Tạo truy vấn tìm kiếm ban đầu
- **ReflectionNode**: Tạo truy vấn tìm kiếm bổ sung sau khi phản ánh

```python
class FirstSearchNode(BaseNode):
    def run(self, input_data: Any, **kwargs) -> Dict[str, str]:
        # Phân tích title và content của đoạn văn
        # Tạo search query phù hợp
        # Chọn search tool tối ưu
        return {
            "search_query": search_query,
            "search_tool": search_tool,
            "reasoning": reasoning
        }
```

##### SummaryNode (summary_node.py)

Tạo và cập nhật nội dung đoạn văn:

- **FirstSummaryNode**: Tạo tóm tắt ban đầu (800-1200 từ)
- **ReflectionSummaryNode**: Cập nhật và mở rộng tóm tắt (1000-1500 từ)

```python
class FirstSummaryNode(StateMutationNode):
    def run(self, input_data: Any, **kwargs) -> str:
        # Xử lý search results
        # Tạo nội dung đoạn văn
        # Tích hợp sentiment analysis
        return processed_response
```

##### ReportFormattingNode (formatting_node.py)

Định dạng báo cáo cuối cùng:

```python
class ReportFormattingNode(BaseNode):
    def run(self, input_data: Any, **kwargs) -> str:
        # Kết hợp tất cả các đoạn văn
        # Định dạng markdown chuyên nghiệp
        return formatted_report
```

#### 3. State Management (state.py)

Quản lý trạng thái của toàn bộ quá trình nghiên cứu:

```python
@dataclass
class State:
    query: str = ""                           # Truy vấn gốc
    report_title: str = ""                    # Tiêu đề báo cáo
    paragraphs: List[Paragraph] = field(default_factory=list)
    final_report: str = ""                    # Báo cáo cuối cùng
    is_completed: bool = False                # Trạng thái hoàn thành
```

##### Data Classes

- **Search**: Lưu trữ thông tin tìm kiếm đơn lẻ
- **Research**: Theo dõi quá trình nghiên cứu của đoạn văn
- **Paragraph**: Quản lý trạng thái của từng đoạn văn

#### 4. Tools Integration

##### MediaCrawlerDB (tools/search.py)

Cung cấp 6 công cụ tìm kiếm chuyên dụng:

1. **search_hot_content** - Tìm nội dung hot theo thời gian
2. **search_topic_globally** - Tìm kiếm toàn cục
3. **search_topic_by_date** - Tìm kiếm theo khoảng thời gian
4. **get_comments_for_topic** - Lấy bình luận cho chủ đề
5. **search_topic_on_platform** - Tìm kiếm trên nền tảng cụ thể
6. **analyze_sentiment** - Phân tích cảm xúc

```python
class MediaCrawlerDB:
    def search_topic_globally(self, topic: str, limit_per_table: int = 100):
        # Tìm kiếm trên tất cả các bảng
        # Trả về kết quả đã định dạng
        return DBResponse("search_topic_globally", params, results)
```

##### Keyword Optimizer (tools/keyword_optimizer.py)

Tối ưu hóa từ khóa tìm kiếm:

```python
class KeywordOptimizer:
    def optimize_keywords(self, original_query: str, context: str):
        # Phân tích truy vấn gốc
        # Tạo các biến thể tối ưu
        return KeywordOptimizationResponse(
            optimized_keywords=[...],
            reasoning="..."
        )
```

##### Sentiment Analyzer (tools/sentiment_analyzer.py)

Phân tích cảm xúc đa ngôn ngữ:

```python
class WeiboMultilingualSentimentAnalyzer:
    def analyze_query_results(self, query_results: List[Dict], text_field: str):
        # Phân tích cảm xúc 22 ngôn ngữ
        # Trả về kết quả chi tiết
        return {
            "sentiment_analysis": {
                "overall_sentiment": "positive",
                "confidence": 0.85,
                "language_distribution": {...}
            }
        }
```

#### 5. Prompt Engineering (prompts/prompts.py)

Định nghĩa các system prompts cho từng giai đoạn:

##### Report Structure Prompt

```python
SYSTEM_PROMPT_REPORT_STRUCTURE = f"""
Bạn是一位专业的舆情分析师和报告架构师。给定一个查询，你需要规划一个全面、深入的舆情分析报告结构。

**报告规划要求：**
1. **段落数量**：设计5个核心段落
2. **内容丰富度**：每个段落应该包含多个子话题和分析维度
3. **逻辑结构**：从宏观到微观、从现象到本质的递进式分析
...
"""
```

##### Search Prompt

```python
SYSTEM_PROMPT_FIRST_SEARCH = f"""
你是一位专业的舆情分析师。你将获得报告中的一个段落...

你可以使用以下6种专业的本地舆情数据库查询工具来挖掘真实的民意和公众观点：

1. **search_hot_content** - 查找热点内容工具
2. **search_topic_globally** - 全局话题搜索工具
...
"""
```

## Luồng hoạt động Chi tiết

### 1. Khởi tạo Agent

```python
# Tạo agent với cấu hình
agent = DeepSearchAgent(config)

# Khởi tạo các thành phần
self.llm_client = LLMClient(
    api_key=config.INSIGHT_ENGINE_API_KEY,
    model_name=config.INSIGHT_ENGINE_MODEL_NAME,
    base_url=config.INSIGHT_ENGINE_BASE_URL
)

self.search_agency = MediaCrawlerDB()
self.sentiment_analyzer = multilingual_sentiment_analyzer
```

### 2. Quy trình Nghiên cứu Chính

#### Bước 1: Tạo Cấu trúc Báo cáo

```python
def _generate_report_structure(self, query: str):
    report_structure_node = ReportStructureNode(self.llm_client, query)
    self.state = report_structure_node.mutate_state(state=self.state)
    
    # Kết quả: 5-7 đoạn văn với title và content
```

#### Bước 2: Xử lý Từng Đoạn văn

```python
def _process_paragraphs(self):
    for i in range(total_paragraphs):
        # Tìm kiếm và tóm tắt ban đầu
        self._initial_search_and_summary(i)
        
        # Vòng lặp phản ánh
        self._reflection_loop(i)
        
        # Đánh dấu hoàn thành
        self.state.paragraphs[i].research.mark_completed()
```

##### Initial Search and Summary

```python
def _initial_search_and_summary(self, paragraph_index: int):
    # 1. Tạo search query
    search_output = self.first_search_node.run(search_input)
    
    # 2. Thực thi search
    search_response = self.execute_search_tool(
        search_output["search_tool"], 
        search_output["search_query"]
    )
    
    # 3. Tạo summary
    self.state = self.first_summary_node.mutate_state(
        summary_input, self.state, paragraph_index
    )
```

##### Reflection Loop

```python
def _reflection_loop(self, paragraph_index: int):
    for reflection_i in range(self.config.MAX_REFLECTIONS):
        # 1. Phản ánh và tạo search query mới
        reflection_output = self.reflection_node.run(reflection_input)
        
        # 2. Thực thi search bổ sung
        search_response = self.execute_search_tool(
            reflection_output["search_tool"],
            reflection_output["search_query"]
        )
        
        # 3. Cập nhật summary
        self.state = self.reflection_summary_node.mutate_state(
            reflection_summary_input, self.state, paragraph_index
        )
```

#### Bước 3: Tạo Báo cáo Cuối cùng

```python
def _generate_final_report(self) -> str:
    # Chuẩn bị dữ liệu báo cáo
    report_data = []
    for paragraph in self.state.paragraphs:
        report_data.append({
            "title": paragraph.title,
            "paragraph_latest_state": paragraph.research.latest_summary
        })
    
    # Định dạng báo cáo
    final_report = self.report_formatting_node.run(report_data)
    return final_report
```

### 3. Tích hợp Công cụ Tìm kiếm

#### Execute Search Tool

```python
def execute_search_tool(self, tool_name: str, query: str, **kwargs):
    # Tối ưu hóa từ khóa
    optimized_response = keyword_optimizer.optimize_keywords(
        original_query=query,
        context=f"sử dụng{tool_name}工具进行查询"
    )
    
    # Thực thi search với từ khóa tối ưu
    all_results = []
    for keyword in optimized_response.optimized_keywords:
        response = self.search_agency.search_topic_globally(keyword)
        all_results.extend(response.results)
    
    # Phân tích cảm xúc
    if kwargs.get("enable_sentiment", True):
        sentiment_analysis = self._perform_sentiment_analysis(all_results)
        integrated_response.parameters["sentiment_analysis"] = sentiment_analysis
    
    return integrated_response
```

### 4. Phân tích Cảm xúc

```python
def _perform_sentiment_analysis(self, results: List):
    # Khởi tạo analyzer nếu cần
    if not self.sentiment_analyzer.is_initialized:
        self.sentiment_analyzer.initialize()
    
    # Chuyển đổi kết quả sang định dạng phù hợp
    results_dict = []
    for result in results:
        results_dict.append({
            "content": result.title_or_content,
            "platform": result.platform,
            "author": result.author_nickname,
            # ...
        })
    
    # Thực thi phân tích
    sentiment_analysis = self.sentiment_analyzer.analyze_query_results(
        query_results=results_dict,
        text_field="content",
        min_confidence=0.5
    )
    
    return sentiment_analysis
```

## Các Tính năng Nâng cao

### 1. Tối ưu hóa Từ khóa Thông minh

- **Phân tích ngữ cảnh**: Hiểu bối cảnh sử dụng từ khóa
- **Tạo biến thể**: Sinh ra nhiều phiên bản từ khóa tối ưu
- **Học từ kết quả**: Cải thiện dựa trên hiệu quả tìm kiếm

### 2. Xử lý JSON Thông minh

- **Multi-line JSON**: Xử lý JSON bị chia cắt qua nhiều dòng
- **Auto-repair**: Tự động sửa lỗi JSON phổ biến
- **Schema validation**: Đảm bảo JSON đúng cấu trúc

### 3. Error Handling và Recovery

```python
def process_output(self, output: str) -> Dict[str, str]:
    try:
        # Thử parse JSON
        result = json.loads(cleaned_output)
    except JSONDecodeError:
        # Thử sửa JSON
        fixed_json = fix_incomplete_json(cleaned_output)
        if fixed_json:
            result = json.loads(fixed_json)
        else:
            # Sử dụng fallback
            return self._get_default_search_query()
```

### 4. Tích hợp ForumEngine

```python
# Đọc host speech từ forum
if FORUM_READER_AVAILABLE:
    try:
        host_speech = get_latest_host_speech()
        if host_speech:
            data['host_speech'] = host_speech
            formatted_host = format_host_speech_for_prompt(host_speech)
            message = formatted_host + "\n" + message
    except Exception as e:
        logger.exception(f"读取HOST发言失败: {str(e)}")
```

## Cấu hình và Tùy chỉnh

### Settings Structure

```python
class Settings(BaseSettings):
    # LLM Configuration
    INSIGHT_ENGINE_API_KEY: str
    INSIGHT_ENGINE_MODEL_NAME: str
    INSIGHT_ENGINE_BASE_URL: str
    
    # Search Configuration
    DEFAULT_SEARCH_TOPIC_GLOBALLY_LIMIT_PER_TABLE: int = 100
    DEFAULT_SEARCH_TOPIC_BY_DATE_LIMIT_PER_TABLE: int = 100
    DEFAULT_GET_COMMENTS_FOR_TOPIC_LIMIT: int = 500
    
    # Processing Configuration
    MAX_REFLECTIONS: int = 2
    MAX_SEARCH_RESULTS_FOR_LLM: int = 10
    MAX_CONTENT_LENGTH: int = 10000
    
    # Output Configuration
    OUTPUT_DIR: str = "insight_engine_streamlit_reports"
    SAVE_INTERMEDIATE_STATES: bool = True
```

### Tùy chỉnh Prompts

Các prompt có thể tùy chỉnh trong `prompts/prompts.py`:

- **SYSTEM_PROMPT_REPORT_STRUCTURE**: Prompt tạo cấu trúc báo cáo
- **SYSTEM_PROMPT_FIRST_SEARCH**: Prompt tạo truy vấn tìm kiếm
- **SYSTEM_PROMPT_FIRST_SUMMARY**: Prompt tạo tóm tắt
- **SYSTEM_PROMPT_REFLECTION**: Prompt phản ánh
- **SYSTEM_PROMPT_REFLECTION_SUMMARY**: Prompt tóm tắt phản ánh
- **SYSTEM_PROMPT_REPORT_FORMATTING**: Prompt định dạng báo cáo

## Hiệu suất và Tối ưu

### 1. Database Optimization

- **Connection pooling**: Sử dụng connection pool hiệu quả
- **Query optimization**: Tối ưu hóa câu lệnh SQL
- **Result caching**: Cache kết quả tìm kiếm

### 2. Memory Management

- **Streaming responses**: Sử dụng streaming để giảm memory usage
- **Batch processing**: Xử lý kết quả theo batch
- **Garbage collection**: Dọn dẹp memory định kỳ

### 3. Processing Optimization

- **Parallel processing**: Xử lý song song các đoạn văn
- **Async operations**: Sử dụng async cho I/O operations
- **Smart batching**: Gom nhóm các operation tương tự

## Tích hợp với Hệ thống

### 1. Streamlit Integration

```python
# SingleEngineApp/insight_engine_streamlit_app.py
import streamlit as st
from InsightEngine import create_agent

@st.cache_resource
def get_agent():
    return create_agent()

def main():
    agent = get_agent()
    
    if st.button("Bắt đầu nghiên cứu"):
        with st.spinner("Đang nghiên cứu..."):
            report = agent.research(query, save_report=True)
        st.markdown(report)
```

### 2. ForumEngine Integration

InsightEngine ghi log structured để ForumEngine có thể xử lý:

```python
# Log format cho ForumEngine
logger.info(f"[FirstSummaryNode] 正在生成首次段落总结")
logger.info(f"清理后的输出: {json.dumps(summary_data)}")
```

### 3. ReportEngine Integration

InsightEngine cung cấp báo cáo cho ReportEngine:

```python
# ReportEngine đọc báo cáo từ InsightEngine
def load_input_files(self, file_paths: Dict[str, str]):
    with open(file_paths['insight'], 'r', encoding='utf-8') as f:
        insight_report = f.read()
    content['reports'].append(insight_report)
```

## Troubleshooting và Debugging

### Các Vấn đề Thường gặp

1. **Database Connection**: Kiểm tra kết nối database
2. **LLM API**: Xác thực API key và rate limits
3. **Memory Issues**: Monitor memory usage
4. **JSON Parsing**: Kiểm tra format JSON output

### Debug Tools

```python
# Debug mode
if self.config.DEBUG_MODE:
    logger.info(f"Debug: search_query={search_query}")
    logger.info(f"Debug: search_results={len(search_results)}")

# State inspection
def inspect_state(self):
    return {
        "paragraphs": len(self.state.paragraphs),
        "completed": self.state.get_completed_paragraphs_count(),
        "progress": self.state.get_progress_summary()
    }
```

## Kết luận

InsightEngine là một động cơ nghiên cứu sâu mạnh mẽ, kết hợp:

- **Công cụ tìm kiếm chuyên biệt** cho cơ sở dữ liệu địa phương
- **Phân tích cảm xúc đa ngôn ngữ** 22 ngôn ngữ
- **Tối ưu hóa từ khóa thông minh**
- **Quy trình nghiên cứu lặp tự động**
- **Tích hợp liền mạch** với các thành phần hệ thống

Với kiến trúc module hóa và khả năng tùy chỉnh cao, InsightEngine cung cấp khả năng phân tích dư luận xã hội sâu sắc và toàn diện, là một trong ba trụ cột chính của hệ thống BettaFish.