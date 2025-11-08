# Tổng quan Kiến trúc MediaEngine

## Giới thiệu

MediaEngine là một trong ba động cơ nghiên cứu chính của hệ thống BettaFish, chuyên biệt trong việc phân tích nội dung đa phương tiện thông qua công cụ tìm kiếm Bocha AI Search. MediaEngine được thiết kế để thu thập, phân tích và tổng hợp thông tin từ nhiều nguồn đa phương tiện khác nhau bao gồm văn bản, hình ảnh, dữ liệu có cấu trúc và các nội dung media khác, tạo ra các báo cáo phân tích toàn diện và đa chiều.

## Mục đích Thiết kế

MediaEngine được thiết kế để:

1. **Tìm kiếm Đa phương tiện**: Truy cập và phân tích thông tin từ nhiều nguồn media khác nhau
2. **Phân tích Nội dung Đa dạng**: Xử lý văn bản, hình ảnh, dữ liệu có cấu trúc cùng lúc
3. **Tích hợp AI Search**: Sử dụng Bocha AI Search để có kết quả tìm kiếm thông minh
4. **Tạo Báo cáo Đa chiều**: Sinh ra các báo cáo phân tích tích hợp nhiều loại media
5. **Phản ánh Sâu**: Thực hiện các vòng lặp nghiên cứu để đạt hiểu biết sâu sắc

## Kiến trúc Chi tiết

### Cấu trúc Thư mục

```
MediaEngine/
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
│   └── search.py                  # BochaMultimodalSearch tools
├── prompts/                       # System prompts
│   ├── __init__.py
│   └── prompts.py                 # All prompt definitions
└── utils/                         # Utilities
    ├── __init__.py
    └── text_processing.py         # Text processing utilities
```

### Các Thành phần Chính

#### 1. DeepSearchAgent (agent.py)

Là thành phần trung tâm điều phối toàn bộ quá trình nghiên cứu đa phương tiện:

```python
class DeepSearchAgent:
    def __init__(self, config: Optional[Settings] = None):
        self.config = config or settings
        self.llm_client = self._initialize_llm()
        self.search_agency = BochaMultimodalSearch(
            api_key=(self.config.BOCHA_API_KEY or self.config.BOCHA_WEB_SEARCH_API_KEY)
        )
        self._initialize_nodes()
        self.state = State()
```

**Các phương thức chính:**
- `research(query, save_report=True)` - Thực hiện nghiên cứu đa phương tiện hoàn chỉnh
- `execute_search_tool(tool_name, query, **kwargs)` - Thực thi công cụ tìm kiếm đa phương tiện
- `get_progress_summary()` - Lấy thông tin tiến độ nghiên cứu

#### 2. BochaMultimodalSearch (tools/search.py)

Cung cấp 5 công cụ tìm kiếm đa phương tiện chuyên dụng:

##### Data Structures

```python
@dataclass
class WebpageResult:
    """Kết quả tìm kiếm webpage"""
    name: str
    url: str
    snippet: str
    display_url: Optional[str] = None
    date_last_crawled: Optional[str] = None

@dataclass
class ImageResult:
    """Kết quả tìm kiếm hình ảnh"""
    name: str
    content_url: str
    host_page_url: Optional[str] = None
    thumbnail_url: Optional[str] = None
    width: Optional[int] = None
    height: Optional[int] = None

@dataclass
class ModalCardResult:
    """Kết quả dữ liệu có cấu trúc (Modal Card)"""
    card_type: str  # weather_china, stock, baike_pro, medical_common
    content: Dict[str, Any]

@dataclass
class BochaResponse:
    """Response hoàn chỉnh từ Bocha API"""
    query: str
    conversation_id: Optional[str] = None
    answer: Optional[str] = None  # AI summary
    follow_ups: List[str] = field(default_factory=list)
    webpages: List[WebpageResult] = field(default_factory=list)
    images: List[ImageResult] = field(default_factory=list)
    modal_cards: List[ModalCardResult] = field(default_factory=list)
```

##### Search Tools

1. **comprehensive_search** - Tìm kiếm toàn diện
```python
def comprehensive_search(self, query: str, max_results: int = 10) -> BochaResponse:
    """Tìm kiếm toàn diện: trả về webpage, hình ảnh, AI summary, follow-up questions"""
    return self._search_internal(
        query=query,
        count=max_results,
        answer=True  # Bật AI summary
    )
```

2. **web_search_only** - Tìm kiếm webpage thuần túy
```python
def web_search_only(self, query: str, max_results: int = 15) -> BochaResponse:
    """Tìm kiếm webpage: không yêu cầu AI summary, nhanh hơn"""
    return self._search_internal(
        query=query,
        count=max_results,
        answer=False  # Tắt AI summary
    )
```

3. **search_for_structured_data** - Tìm kiếm dữ liệu có cấu trúc
```python
def search_for_structured_data(self, query: str) -> BochaResponse:
    """Chuyên dụng cho weather, stock, exchange rate, encyclopedia"""
    return self._search_internal(
        query=query,
        count=5,
        answer=True
    )
```

4. **search_last_24_hours** - Tìm kiếm 24 giờ gần nhất
```python
def search_last_24_hours(self, query: str) -> BochaResponse:
    """Tìm kiếm thông tin 24 giờ gần nhất cho breaking news"""
    return self._search_internal(query=query, freshness='oneDay', answer=True)
```

5. **search_last_week** - Tìm kiếm tuần gần nhất
```python
def search_last_week(self, query: str) -> BochaResponse:
    """Tìm kiếm thông tin tuần gần nhất cho weekly summary"""
    return self._search_internal(query=query, freshness='oneWeek', answer=True)
```

#### 3. Processing Nodes

##### BaseNode (base_node.py)

Tương tự InsightEngine, nhưng được tùy chỉnh cho đa phương tiện:

```python
class BaseNode(ABC):
    def __init__(self, llm_client: LLMClient, node_name: str = ""):
        self.llm_client = llm_client
        self.node_name = node_name or self.__class__.__name__
    
    @abstractmethod
    def run(self, input_data: Any, **kwargs) -> Any:
        pass
```

##### SearchNode (search_node.py)

Tạo các truy vấn tìm kiếm đa phương tiện:

- **FirstSearchNode**: Tạo truy vấn ban đầu với 5 công cụ Bocha
- **ReflectionNode**: Tạo truy vấn bổ sung cho nội dung đa phương tiện

```python
class FirstSearchNode(BaseNode):
    def run(self, input_data: Any, **kwargs) -> Dict[str, str]:
        # Phân tích title và content của đoạn văn
        # Chọn tool phù hợp: comprehensive_search, web_search_only, etc.
        # Tạo search query tối ưu cho đa phương tiện
        return {
            "search_query": search_query,
            "search_tool": search_tool,
            "reasoning": reasoning
        }
```

##### SummaryNode (summary_node.py)

Tạo nội dung phân tích đa phương tiện:

- **FirstSummaryNode**: Tạo phân tích ban đầu (800-1200 từ)
- **ReflectionSummaryNode**: Cập nhật và mở rộng phân tích (1000-1500 từ)

```python
class FirstSummaryNode(StateMutationNode):
    def run(self, input_data: Any, **kwargs) -> str:
        # Xử lý đa dạng search results: webpages, images, modal cards
        # Tích hợp AI summary từ Bocha
        # Tạo phân tích đa chiều
        return multimedia_analysis
```

#### 4. State Management (state.py)

Tương tự InsightEngine nhưng được tối ưu cho đa phương tiện:

```python
@dataclass
class State:
    query: str = ""
    report_title: str = ""
    paragraphs: List[Paragraph] = field(default_factory=list)
    final_report: str = ""
    is_completed: bool = False
    
    # Thông tin đa phương tiện bổ sung
    multimedia_sources: Dict[str, Any] = field(default_factory=dict)
    image_analysis: List[Dict] = field(default_factory=list)
```

#### 5. Prompt Engineering (prompts/prompts.py)

Các prompt được thiết kế đặc biệt cho phân tích đa phương tiện:

##### First Search Prompt

```python
SYSTEM_PROMPT_FIRST_SEARCH = f"""
你是一位深度研究助手。你将获得报告中的一个段落...

你可以使用以下5种专业的多模态搜索工具：

1. **comprehensive_search** - 全面综合搜索工具
   - 适用于：一般性的研究需求，需要完整信息时
   - 特点：返回网页、图片、AI总结、追问建议和可能的结构化数据

2. **web_search_only** - 纯网页搜索工具
   - 适用于：只需要网页链接和摘要，不需要AI分析时
   - 特点：速度更快，成本更低，只返回网页结果

3. **search_for_structured_data** - 结构化数据查询工具
   - 适用于：查询天气、股票、汇率、百科定义等结构化信息时
   - 特点：专门用于触发"模态卡"的查询，返回结构化数据

4. **search_last_24_hours** - 24小时内信息搜索工具
   - 适用于：需要了解最新动态、突发事件时
   - 特点：只搜索过去24小时内发布的内容

5. **search_last_week** - 本周信息搜索工具
   - 适用于：需要了解近期发展趋势时
   - 特点：搜索过去一周内的主要报道
...
"""
```

##### First Summary Prompt

```python
SYSTEM_PROMPT_FIRST_SUMMARY = f"""
你是一位专业的多媒体内容分析师和深度报告撰写专家...

**撰写标准和多模态内容整合要求：**

1. **多源信息整合层次**：
   - **网页内容分析**：详细分析网页搜索结果中的文字信息、数据、观点
   - **图片信息解读**：深入分析相关图片所传达的信息、情感、视觉元素
   - **AI总结整合**：利用AI总结信息，提炼关键观点和趋势
   - **结构化数据应用**：充分利用天气、股票、百科等结构化信息

2. **内容结构化组织**：
   ```
   ## 综合信息概览
   [多种信息源的核心发现]
   
   ## 文本内容深度分析
   [网页、文章内容的详细分析]
   
   ## 视觉信息解读
   [图片、多媒体内容的分析]
   
   ## 数据综合分析
   [各类数据的整合分析]
   
   ## 多维度洞察
   [基于多种信息源的深度洞察]
   ```
...
"""
```

## Luồng hoạt động Chi tiết

### 1. Khởi tạo Agent

```python
# Tạo agent với cấu hình Bocha
agent = DeepSearchAgent(config)

# Khởi tạo BochaMultimodalSearch
self.search_agency = BochaMultimodalSearch(
    api_key=config.BOCHA_WEB_SEARCH_API_KEY
)
```

### 2. Quy trình Nghiên cứu Đa phương tiện

#### Bước 1: Tạo Cấu trúc Báo cáo

```python
def _generate_report_structure(self, query: str):
    report_structure_node = ReportStructureNode(self.llm_client, query)
    self.state = report_structure_node.mutate_state(state=self.state)
    
    # Kết quả: 5 đoạn văn với title và content
```

#### Bước 2: Xử lý Từng Đoạn văn

##### Initial Search and Summary

```python
def _initial_search_and_summary(self, paragraph_index: int):
    # 1. Tạo search query
    search_output = self.first_search_node.run(search_input)
    
    # 2. Thực thi search với tool đa phương tiện
    search_response = self.execute_search_tool(
        search_output["search_tool"], 
        search_output["search_query"]
    )
    
    # 3. Xử lý đa dạng kết quả
    search_results = []
    if search_response.webpages:
        for result in search_response.webpages:
            search_results.append({
                'title': result.name,
                'url': result.url,
                'content': result.snippet,
                'type': 'webpage'
            })
    
    if search_response.images:
        for result in search_response.images:
            search_results.append({
                'title': result.name,
                'url': result.content_url,
                'content': f"Image: {result.name}",
                'type': 'image',
                'thumbnail': result.thumbnail_url
            })
    
    # 4. Tạo summary đa phương tiện
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
        
        # 3. Cập nhật summary với nội dung đa phương tiện mới
        self.state = self.reflection_summary_node.mutate_state(
            reflection_summary_input, self.state, paragraph_index
        )
```

### 3. Xử lý Kết quả Đa phương tiện

#### Parse Bocha Response

```python
def _parse_search_response(self, response_dict: Dict[str, Any], query: str):
    final_response = BochaResponse(query=query)
    
    messages = response_dict.get('messages', [])
    for msg in messages:
        if msg.get('role') != 'assistant':
            continue
            
        msg_type = msg.get('type')
        content_type = msg.get('content_type')
        content_data = json.loads(msg.get('content', '{}'))
        
        if msg_type == 'answer' and content_type == 'text':
            final_response.answer = content_data
        elif msg_type == 'follow_up' and content_type == 'text':
            final_response.follow_ups.append(content_data)
        elif msg_type == 'source':
            if content_type == 'webpage':
                # Xử lý webpage results
                web_results = content_data.get('value', [])
                for item in web_results:
                    final_response.webpages.append(WebpageResult(...))
            elif content_type == 'image':
                # Xử lý image results
                final_response.images.append(ImageResult(...))
            else:
                # Xử lý modal cards (dữ liệu có cấu trúc)
                final_response.modal_cards.append(ModalCardResult(...))
    
    return final_response
```

#### Xử lý Modal Cards

```python
# Ví dụ xử lý weather modal card
if response.modal_cards:
    for card in response.modal_cards:
        if card.card_type == 'weather_china':
            weather_data = card.content
            # Trích xuất thông tin thời tiết
            temperature = weather_data.get('temperature')
            humidity = weather_data.get('humidity')
            wind_speed = weather_data.get('windSpeed')
            
            # Tích hợp vào phân tích
            weather_analysis = f"Dựa trên dữ liệu thời tiết, nhiệt độ {temperature}°C, độ ẩm {humidity}%"
```

### 4. Tạo Báo cáo Đa phương tiện

#### Report Formatting

```python
def _generate_final_report(self) -> str:
    # Chuẩn bị dữ liệu đa phương tiện
    report_data = []
    for paragraph in self.state.paragraphs:
        report_data.append({
            "title": paragraph.title,
            "paragraph_latest_state": paragraph.research.latest_summary
        })
    
    # Định dạng báo cáo đa phương tiện
    final_report = self.report_formatting_node.run(report_data)
    return final_report
```

## Các Tính năng Nâng cao

### 1. Modal Cards Processing

MediaEngine có khả năng xử lý các loại modal cards khác nhau:

- **weather_china**: Thông tin thời tiết Trung Quốc
- **stock**: Dữ liệu chứng khoán
- **baike_pro**: Bách khoa toàn thư
- **medical_common**: Thông tin y tế phổ thông

```python
def process_modal_cards(self, modal_cards: List[ModalCardResult]):
    processed_data = {}
    
    for card in modal_cards:
        card_type = card.card_type
        content = card.content
        
        if card_type == 'weather_china':
            processed_data['weather'] = {
                'temperature': content.get('temperature'),
                'humidity': content.get('humidity'),
                'weather_desc': content.get('weatherDesc')
            }
        elif card_type == 'stock':
            processed_data['stock'] = {
                'price': content.get('currentPrice'),
                'change': content.get('changePercent'),
                'volume': content.get('volume')
            }
    
    return processed_data
```

### 2. Multi-source Content Integration

```python
def integrate_multimedia_content(self, search_response: BochaResponse):
    integrated_content = {
        'text_analysis': self._analyze_webpages(search_response.webpages),
        'visual_analysis': self._analyze_images(search_response.images),
        'structured_data': self._analyze_modal_cards(search_response.modal_cards),
        'ai_summary': search_response.answer,
        'follow_up_questions': search_response.follow_ups
    }
    
    return integrated_content
```

### 3. Image Analysis

```python
def _analyze_images(self, images: List[ImageResult]):
    image_analysis = []
    
    for image in images:
        analysis = {
            'name': image.name,
            'url': image.content_url,
            'thumbnail': image.thumbnail_url,
            'dimensions': f"{image.width}x{image.height}" if image.width and image.height else "Unknown",
            'description': f"Hình ảnh về {image.name}",
            'host_page': image.host_page_url
        }
        image_analysis.append(analysis)
    
    return image_analysis
```

### 4. Error Handling và Retry

```python
@with_graceful_retry(SEARCH_API_RETRY_CONFIG, default_return=BochaResponse(query="搜索失败"))
def _search_internal(self, **kwargs) -> BochaResponse:
    try:
        response = requests.post(self.BOCHA_BASE_URL, headers=self._headers, json=payload, timeout=30)
        response.raise_for_status()
        
        response_dict = response.json()
        if response_dict.get("code") != 200:
            logger.error(f"API返回错误: {response_dict.get('msg', '未知错误')}")
            return BochaResponse(query=query)
        
        return self._parse_search_response(response_dict, query)
        
    except requests.exceptions.RequestException as e:
        logger.exception(f"搜索时发生网络错误: {str(e)}")
        raise e
```

## Cấu hình và Tùy chỉnh

### Settings Structure

```python
class Settings(BaseSettings):
    # Bocha Configuration
    BOCHA_API_KEY: str
    BOCHA_WEB_SEARCH_API_KEY: str
    BOCHA_BASE_URL: str = "https://api.bochaai.com/v1/ai-search"
    
    # Search Configuration
    DEFAULT_MAX_RESULTS: int = 10
    SEARCH_CONTENT_MAX_LENGTH: int = 10000
    
    # Processing Configuration
    MAX_REFLECTIONS: int = 2
    
    # Output Configuration
    OUTPUT_DIR: str = "media_engine_streamlit_reports"
    SAVE_INTERMEDIATE_STATES: bool = True
```

### Tùy chỉnh Search Tools

```python
# Tùy chỉnh các tham số search
search_config = {
    'comprehensive_search': {
        'max_results': 15,
        'include_ai_summary': True
    },
    'web_search_only': {
        'max_results': 20,
        'include_ai_summary': False
    },
    'search_last_24_hours': {
        'freshness': 'oneDay',
        'include_ai_summary': True
    }
}
```

## Hiệu suất và Tối ưu

### 1. API Rate Limiting

```python
import time
from typing import Dict

class RateLimiter:
    def __init__(self, max_requests_per_minute: int = 60):
        self.max_requests = max_requests_per_minute
        self.requests = []
    
    def wait_if_needed(self):
        now = time.time()
        # Remove requests older than 1 minute
        self.requests = [req_time for req_time in self.requests if now - req_time < 60]
        
        if len(self.requests) >= self.max_requests:
            sleep_time = 60 - (now - self.requests[0])
            if sleep_time > 0:
                time.sleep(sleep_time)
        
        self.requests.append(now)
```

### 2. Response Caching

```python
from functools import lru_cache
import hashlib

class SearchCache:
    def __init__(self, max_size: int = 1000):
        self.cache = {}
        self.max_size = max_size
    
    def get_cache_key(self, query: str, tool: str, **kwargs) -> str:
        key_data = f"{query}:{tool}:{sorted(kwargs.items())}"
        return hashlib.md5(key_data.encode()).hexdigest()
    
    def get(self, cache_key: str) -> Optional[BochaResponse]:
        return self.cache.get(cache_key)
    
    def set(self, cache_key: str, response: BochaResponse):
        if len(self.cache) >= self.max_size:
            # Remove oldest entry
            oldest_key = next(iter(self.cache))
            del self.cache[oldest_key]
        
        self.cache[cache_key] = response
```

### 3. Parallel Processing

```python
import concurrent.futures
from typing import List

def parallel_search(self, queries: List[str], tool_name: str) -> List[BochaResponse]:
    with concurrent.futures.ThreadPoolExecutor(max_workers=3) as executor:
        futures = []
        for query in queries:
            future = executor.submit(self.execute_search_tool, tool_name, query)
            futures.append(future)
        
        results = []
        for future in concurrent.futures.as_completed(futures):
            try:
                result = future.result()
                results.append(result)
            except Exception as e:
                logger.exception(f"Parallel search failed: {e}")
        
        return results
```

## Tích hợp với Hệ thống

### 1. Streamlit Integration

```python
# SingleEngineApp/media_engine_streamlit_app.py
import streamlit as st
from MediaEngine import create_agent

@st.cache_resource
def get_agent():
    return create_agent()

def main():
    agent = get_agent()
    
    st.title("MediaEngine - Phân tích Đa phương tiện")
    
    query = st.text_input("Nhập truy vấn nghiên cứu:")
    
    if st.button("Bắt đầu nghiên cứu") and query:
        with st.spinner("Đang nghiên cứu đa phương tiện..."):
            report = agent.research(query, save_report=True)
        
        # Hiển thị kết quả đa phương tiện
        st.markdown(report)
        
        # Hiển thị thống kê đa phương tiện
        if hasattr(agent.state, 'multimedia_sources'):
            st.json(agent.state.multimedia_sources)
```

### 2. ForumEngine Integration

MediaEngine ghi log structured để ForumEngine có thể xử lý:

```python
# Log format cho ForumEngine
logger.info(f"[FirstSummaryNode] 正在生成首次段落总结")
logger.info(f"清理后的输出: {json.dumps(multimedia_summary_data)}")

# Log thông tin đa phương tiện
if search_response.images:
    logger.info(f"Found {len(search_response.images)} images")
    
if search_response.modal_cards:
    logger.info(f"Found {len(search_response.modal_cards)} modal cards")
```

### 3. ReportEngine Integration

MediaEngine cung cấp báo cáo đa phương tiện cho ReportEngine:

```python
# ReportEngine đọc báo cáo từ MediaEngine
def load_input_files(self, file_paths: Dict[str, str]):
    with open(file_paths['media'], 'r', encoding='utf-8') as f:
        media_report = f.read()
    content['reports'].append(media_report)
    
    # Đọc thêm thông tin đa phương tiện nếu có
    media_metadata_file = file_paths.get('media_metadata')
    if media_metadata_file and os.path.exists(media_metadata_file):
        with open(media_metadata_file, 'r', encoding='utf-8') as f:
            content['media_metadata'] = json.load(f)
```

## Troubleshooting và Debugging

### Các Vấn đề Thường gặp

1. **Bocha API Connection**: Kiểm tra API key và rate limits
2. **Modal Card Parsing**: Xác thực format của modal cards
3. **Image URL Processing**: Kiểm tra URL validity
4. **Multi-source Integration**: Debug content merging

### Debug Tools

```python
# Debug mode cho Bocha search
if self.config.DEBUG_MODE:
    logger.info(f"Debug: search_query={search_query}")
    logger.info(f"Debug: search_tool={search_tool}")
    logger.info(f"Debug: response={response_dict}")

# Multimedia content inspection
def inspect_multimedia_content(self, search_response: BochaResponse):
    return {
        "webpages_count": len(search_response.webpages),
        "images_count": len(search_response.images),
        "modal_cards_count": len(search_response.modal_cards),
        "has_ai_summary": bool(search_response.answer),
        "follow_ups_count": len(search_response.follow_ups),
        "modal_card_types": [card.card_type for card in search_response.modal_cards]
    }
```

## Kết luận

MediaEngine là một động cơ nghiên cứu đa phương tiện mạnh mẽ, kết hợp:

- **5 công cụ tìm kiếm chuyên dụng** từ Bocha AI Search
- **Khả năng xử lý đa dạng** nội dung (text, image, structured data)
- **Phân tích đa chiều** thông tin từ nhiều nguồn
- **Tích hợp AI thông minh** cho summarization và follow-up
- **Quy trình nghiên cứu lặp** để đạt hiểu biết sâu sắc

Với kiến trúc được thiết kế đặc biệt cho nội dung đa phương tiện, MediaEngine cung cấp khả năng phân tích toàn diện và sâu sắc, là một trong ba trụ cột chính của hệ thống BettaFish, chuyên biệt trong việc xử lý và phân tích thông tin đa phương tiện từ internet.