# Tổng quan Kiến trúc QueryEngine

## Giới thiệu

QueryEngine là một trong ba động cơ nghiên cứu chính của hệ thống BettaFish, chuyên biệt trong việc phân tích tin tức và thông tin thời sự thông qua công cụ tìm kiếm Tavily News. QueryEngine được thiết kế để thu thập, phân tích và tổng hợp thông tin từ các nguồn tin tức đa dạng, tạo ra các báo cáo phân tích sâu về các sự kiện, xu hướng và vấn đề thời sự.

## Mục đích Thiết kế

QueryEngine được thiết kế để:

1. **Tìm kiếm Tin tức**: Truy cập và phân tích thông tin từ các nguồn tin tức toàn cầu
2. **Phân tích Thời sự**: Theo dõi các sự kiện breaking news và xu hướng thời sự
3. **Kiểm chứng Thông tin**: Đánh giá độ tin cậy và xác thực thông tin
4. **Tạo Báo cáo Tin tức**: Sinh ra các báo cáo phân tích tin tức chuyên nghiệp
5. **Phản ánh Sâu**: Thực hiện các vòng lặp nghiên cứu để đạt hiểu biết sâu sắc

## Kiến trúc Chi tiết

### Cấu trúc Thư mục

```
QueryEngine/
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
│   └── search.py                  # TavilyNewsAgency tools
├── prompts/                       # System prompts
│   ├── __init__.py
│   └── prompts.py                 # All prompt definitions
└── utils/                         # Utilities
    ├── __init__.py
    └── text_processing.py         # Text processing utilities
```

### Các Thành phần Chính

#### 1. DeepSearchAgent (agent.py)

Là thành phần trung tâm điều phối toàn bộ quá trình nghiên cứu tin tức:

```python
class DeepSearchAgent:
    def __init__(self, config: Optional[Settings] = None):
        self.config = config or settings
        self.llm_client = self._initialize_llm()
        self.search_agency = TavilyNewsAgency(api_key=self.config.TAVILY_API_KEY)
        self._initialize_nodes()
        self.state = State()
```

**Các phương thức chính:**
- `research(query, save_report=True)` - Thực hiện nghiên cứu tin tức hoàn chỉnh
- `execute_search_tool(tool_name, query, **kwargs)` - Thực thi công cụ tìm kiếm tin tức
- `_validate_date_format(date_str)` - Kiểm tra định dạng ngày tháng

#### 2. TavilyNewsAgency (tools/search.py)

Cung cấp 6 công cụ tìm kiếm tin tức chuyên dụng:

##### Data Structures

```python
@dataclass
class SearchResult:
    """Kết quả tìm kiếm tin tức với ngày phát hành"""
    title: str
    url: str
    content: str
    score: Optional[float] = None
    raw_content: Optional[str] = None
    published_date: Optional[str] = None  # Ngày phát hành tin tức

@dataclass
class ImageResult:
    """Kết quả tìm kiếm hình ảnh tin tức"""
    url: str
    description: Optional[str] = None

@dataclass
class TavilyResponse:
    """Response hoàn chỉnh từ Tavily API"""
    query: str
    answer: Optional[str] = None  # AI summary
    results: List[SearchResult] = field(default_factory=list)
    images: List[ImageResult] = field(default_factory=list)
    response_time: Optional[float] = None
```

##### Search Tools

1. **basic_search_news** - Tìm kiếm tin tức cơ bản
```python
def basic_search_news(self, query: str, max_results: int = 7) -> TavilyResponse:
    """Tìm kiếm tin tức cơ bản: nhanh, tiêu chuẩn, phù hợp cho hầu hết các trường hợp"""
    return self._search_internal(
        query=query,
        max_results=max_results,
        search_depth="basic",
        include_answer=False
    )
```

2. **deep_search_news** - Phân tích tin tức sâu
```python
def deep_search_news(self, query: str) -> TavilyResponse:
    """Phân tích tin tức sâu: toàn diện, chi tiết nhất với AI summary cao cấp"""
    return self._search_internal(
        query=query, 
        search_depth="advanced", 
        max_results=20, 
        include_answer="advanced"
    )
```

3. **search_news_last_24_hours** - Tin tức 24 giờ qua
```python
def search_news_last_24_hours(self, query: str) -> TavilyResponse:
    """Tìm kiếm tin tức 24 giờ gần nhất cho breaking news"""
    return self._search_internal(query=query, time_range='d', max_results=10)
```

4. **search_news_last_week** - Tin tức tuần qua
```python
def search_news_last_week(self, query: str) -> TavilyResponse:
    """Tìm kiếm tin tức tuần gần nhất cho weekly summary"""
    return self._search_internal(query=query, time_range='w', max_results=10)
```

5. **search_images_for_news** - Tìm kiếm hình ảnh tin tức
```python
def search_images_for_news(self, query: str) -> TavilyResponse:
    """Tìm kiếm hình ảnh liên quan đến tin tức"""
    return self._search_internal(
        query=query, 
        include_images=True, 
        include_image_descriptions=True, 
        max_results=5
    )
```

6. **search_news_by_date** - Tìm kiếm theo khoảng thời gian
```python
def search_news_by_date(self, query: str, start_date: str, end_date: str) -> TavilyResponse:
    """Tìm kiếm tin tức trong khoảng thời gian cụ thể"""
    return self._search_internal(
        query=query, 
        start_date=start_date, 
        end_date=end_date, 
        max_results=15
    )
```

#### 3. Processing Nodes

##### BaseNode (base_node.py)

Tương tự các engine khác, được tùy chỉnh cho tin tức:

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

Tạo các truy vấn tìm kiếm tin tức:

- **FirstSearchNode**: Tạo truy vấn ban đầu với 6 công cụ Tavily
- **ReflectionNode**: Tạo truy vấn bổ sung cho nội dung tin tức

```python
class FirstSearchNode(BaseNode):
    def run(self, input_data: Any, **kwargs) -> Dict[str, str]:
        # Phân tích title và content của đoạn văn
        # Chọn tool phù hợp: basic_search_news, deep_search_news, etc.
        # Xử lý đặc biệt cho search_news_by_date (cần start_date, end_date)
        return {
            "search_query": search_query,
            "search_tool": search_tool,
            "reasoning": reasoning,
            "start_date": start_date,  # Nếu cần
            "end_date": end_date      # Nếu cần
        }
```

##### SummaryNode (summary_node.py)

Tạo nội dung phân tích tin tức:

- **FirstSummaryNode**: Tạo phân tích ban đầu (800-1200 từ)
- **ReflectionSummaryNode**: Cập nhật và mở rộng phân tích (1000-1500 từ)

```python
class FirstSummaryNode(StateMutationNode):
    def run(self, input_data: Any, **kwargs) -> str:
        # Xử lý search results với published_date
        # Tích hợp AI answer từ Tavily
        # Tạo phân tích tin tức chuyên nghiệp
        return news_analysis
```

#### 4. State Management (state.py)

Tương tự các engine khác nhưng được tối ưu cho tin tức:

```python
@dataclass
class State:
    query: str = ""
    report_title: str = ""
    paragraphs: List[Paragraph] = field(default_factory=list)
    final_report: str = ""
    is_completed: bool = False
    
    # Thông tin tin tức bổ sung
    news_sources: Dict[str, Any] = field(default_factory=dict)
    publication_dates: List[str] = field(default_factory=list)
    fact_checking: List[Dict] = field(default_factory=list)
```

#### 5. Prompt Engineering (prompts/prompts.py)

Các prompt được thiết kế đặc biệt cho phân tích tin tức:

##### First Search Prompt

```python
SYSTEM_PROMPT_FIRST_SEARCH = f"""
你是一位深度研究助手。你将获得报告中的一个段落...

你可以使用以下6种专业的新闻搜索工具：

1. **basic_search_news** - 基础新闻搜索工具
   - 适用于：一般性的新闻搜索，不确定需要何种特定搜索时
   - 特点：快速、标准的通用搜索，是最常用的基础工具

2. **deep_search_news** - 深度新闻分析工具
   - 适用于：需要全面深入了解某个主题时
   - 特点：提供最详细的分析结果，包含高级AI摘要

3. **search_news_last_24_hours** - 24小时最新新闻工具
   - 适用于：需要了解最新动态、突发事件时
   - 特点：只搜索过去24小时的新闻

4. **search_news_last_week** - 本周新闻工具
   - 适用于：需要了解近期发展趋势时
   - 特点：搜索过去一周的新闻报道

5. **search_images_for_news** - 图片搜索工具
   - 适用于：需要可视化信息、图片资料时
   - 特点：提供相关图片和图片描述

6. **search_news_by_date** - 按日期范围搜索工具
   - 适用于：需要研究特定历史时期时
   - 特点：可以指定开始和结束日期进行搜索
   - 特殊要求：需要提供start_date和end_date参数，格式为'YYYY-MM-DD'

你的任务是：
1. 根据段落主题选择最合适的搜索工具
2. 制定最佳的搜索查询
3. 如果选择search_news_by_date工具，必须同时提供start_date和end_date参数
4. 解释你的选择理由
5. 仔细核查新闻中的可疑点，破除谣言和误导，尽力还原事件原貌
...
"""
```

##### First Summary Prompt

```python
SYSTEM_PROMPT_FIRST_SUMMARY = f"""
你是一位专业的新闻分析师和深度内容创作专家...

**撰写标准和要求：**

1. **丰富的信息层次**：
   - **事实陈述层**：详细引用新闻报道的具体内容、数据、事件细节
   - **多源验证层**：对比不同新闻源的报道角度和信息差异
   - **数据分析层**：提取并分析相关的数量、时间、地点等关键数据
   - **深度解读层**：分析事件背后的原因、影响和意义

2. **结构化内容组织**：
   ```
   ## 核心事件概述
   [详细的事件描述和关键信息]
   
   ## 多方报道分析
   [不同媒体的报道角度和信息汇总]
   
   ## 关键数据提取
   [重要的数字、时间、地点等数据]
   
   ## 深度背景分析
   [事件的背景、原因、影响分析]
   
   ## 发展趋势判断
   [基于现有信息的趋势分析]
   ```

3. **具体引用要求**：
   - **直接引用**：大量使用引号标注的新闻原文
   - **数据引用**：精确引用报道中的数字、统计数据
   - **多源对比**：展示不同新闻源的表述差异
   - **时间线整理**：按时间顺序整理事件发展脉络
...
"""
```

## Luồng hoạt động Chi tiết

### 1. Khởi tạo Agent

```python
# Tạo agent với cấu hình Tavily
agent = DeepSearchAgent(config)

# Khởi tạo TavilyNewsAgency
self.search_agency = TavilyNewsAgency(api_key=config.TAVILY_API_KEY)
```

### 2. Quy trình Nghiên cứu Tin tức

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
    
    # 2. Xử lý đặc biệt cho search_news_by_date
    search_kwargs = {}
    if search_output.get("search_tool") == "search_news_by_date":
        start_date = search_output.get("start_date")
        end_date = search_output.get("end_date")
        if start_date and end_date:
            if self._validate_date_format(start_date) and self._validate_date_format(end_date):
                search_kwargs["start_date"] = start_date
                search_kwargs["end_date"] = end_date
            else:
                # Chuyển sang basic_search_news nếu date format sai
                search_output["search_tool"] = "basic_search_news"
    
    # 3. Thực thi search
    search_response = self.execute_search_tool(
        search_output["search_tool"], 
        search_output["search_query"],
        **search_kwargs
    )
    
    # 4. Xử lý kết quả tin tức với published_date
    search_results = []
    if search_response and search_response.results:
        for result in search_response.results:
            search_results.append({
                'title': result.title,
                'url': result.url,
                'content': result.content,
                'score': result.score,
                'published_date': result.published_date,  # Ngày phát hành
                'raw_content': result.raw_content
            })
    
    # 5. Tạo summary tin tức
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
        
        # 2. Xử lý date parameters cho reflection
        search_kwargs = {}
        if reflection_output.get("search_tool") == "search_news_by_date":
            start_date = reflection_output.get("start_date")
            end_date = reflection_output.get("end_date")
            if start_date and end_date:
                if self._validate_date_format(start_date) and self._validate_date_format(end_date):
                    search_kwargs["start_date"] = start_date
                    search_kwargs["end_date"] = end_date
                else:
                    reflection_output["search_tool"] = "basic_search_news"
        
        # 3. Thực thi search bổ sung
        search_response = self.execute_search_tool(
            reflection_output["search_tool"],
            reflection_output["search_query"],
            **search_kwargs
        )
        
        # 4. Cập nhật summary
        self.state = self.reflection_summary_node.mutate_state(
            reflection_summary_input, self.state, paragraph_index
        )
```

### 3. Xử lý Kết quả Tin tức

#### Date Validation

```python
def _validate_date_format(self, date_str: str) -> bool:
    """Kiểm tra định dạng ngày tháng YYYY-MM-DD"""
    if not date_str:
        return False
    
    # Kiểm tra format
    pattern = r'^\d{4}-\d{2}-\d{2}$'
    if not re.match(pattern, date_str):
        return False
    
    # Kiểm tra ngày hợp lệ
    try:
        datetime.strptime(date_str, '%Y-%m-%d')
        return True
    except ValueError:
        return False
```

#### Parse Tavily Response

```python
def _search_internal(self, **kwargs) -> TavilyResponse:
    try:
        kwargs['topic'] = 'general'
        api_params = {k: v for k, v in kwargs.items() if v is not None}
        response_dict = self._client.search(**api_params)
        
        search_results = [
            SearchResult(
                title=item.get('title'),
                url=item.get('url'),
                content=item.get('content'),
                score=item.get('score'),
                raw_content=item.get('raw_content'),
                published_date=item.get('published_date')  # Ngày phát hành
            ) for item in response_dict.get('results', [])
        ]
        
        image_results = [
            ImageResult(
                url=item.get('url'), 
                description=item.get('description')
            ) for item in response_dict.get('images', [])
        ]

        return TavilyResponse(
            query=response_dict.get('query'), 
            answer=response_dict.get('answer'),
            results=search_results, 
            images=image_results,
            response_time=response_dict.get('response_time')
        )
    except Exception as e:
        print(f"搜索时发生错误: {str(e)}")
        raise e
```

### 4. Tạo Báo cáo Tin tức

#### Report Formatting

```python
def _generate_final_report(self) -> str:
    # Chuẩn bị dữ liệu tin tức
    report_data = []
    for paragraph in self.state.paragraphs:
        report_data.append({
            "title": paragraph.title,
            "paragraph_latest_state": paragraph.research.latest_summary
        })
    
    # Định dạng báo cáo tin tức chuyên nghiệp
    final_report = self.report_formatting_node.run(report_data)
    return final_report
```

## Các Tính năng Nâng cao

### 1. Fact Checking

QueryEngine có khả năng kiểm chứng thông tin:

```python
def fact_check_news(self, search_results: List[SearchResult]) -> Dict[str, Any]:
    """Kiểm chứng độ tin cậy của tin tức"""
    fact_checking = {
        'source_credibility': {},
        'content_consistency': {},
        'publication_analysis': {}
    }
    
    for result in search_results:
        # Phân tích độ tin cậy của nguồn
        source_domain = urlparse(result.url).netloc
        fact_checking['source_credibility'][source_domain] = self._analyze_source_credibility(source_domain)
        
        # Kiểm tra tính nhất quán của nội dung
        fact_checking['content_consistency'][result.url] = self._check_content_consistency(result.content)
        
        # Phân tích ngày phát hành
        if result.published_date:
            fact_checking['publication_analysis'][result.url] = {
                'published_date': result.published_date,
                'timeliness': self._analyze_timeliness(result.published_date)
            }
    
    return fact_checking
```

### 2. Multi-source Verification

```python
def verify_multi_source(self, query: str, search_results: List[SearchResult]) -> Dict[str, Any]:
    """Xác thực thông tin từ nhiều nguồn"""
    verification = {
        'common_facts': [],
        'conflicting_reports': [],
        'source_diversity': {},
        'timeline_consistency': {}
    }
    
    # Phân tích các sự kiện chung
    common_keywords = self._extract_common_keywords(search_results)
    verification['common_facts'] = common_keywords
    
    # Tìm các báo cáo mâu thuẫn
    conflicting_info = self._identify_conflicts(search_results)
    verification['conflicting_reports'] = conflicting_info
    
    # Phân tích đa dạng nguồn
    source_domains = [urlparse(result.url).netloc for result in search_results]
    verification['source_diversity'] = {
        'unique_sources': len(set(source_domains)),
        'source_types': self._categorize_sources(source_domains)
    }
    
    return verification
```

### 3. Breaking News Detection

```python
def detect_breaking_news(self, query: str) -> Dict[str, Any]:
    """Phát hiện breaking news"""
    # Sử dụng search_last_24_hours để tìm tin tức mới
    recent_news = self.search_news_last_24_hours(query)
    
    breaking_indicators = {
        'urgency_keywords': ['紧急', '突发', '即时', 'breaking', 'urgent'],
        'high_frequency': len(recent_news.results) > 10,
        'source_multiplicity': len(set([urlparse(r.url).netloc for r in recent_news.results])) > 5,
        'time_proximity': self._analyze_time_proximity(recent_news.results)
    }
    
    breaking_score = sum([
        breaking_indicators['urgency_keywords'] * 0.3,
        breaking_indicators['high_frequency'] * 0.2,
        breaking_indicators['source_multiplicity'] * 0.3,
        breaking_indicators['time_proximity'] * 0.2
    ])
    
    return {
        'is_breaking': breaking_score > 0.7,
        'score': breaking_score,
        'recent_articles': len(recent_news.results),
        'indicators': breaking_indicators
    }
```

### 4. Error Handling và Retry

```python
@with_graceful_retry(SEARCH_API_RETRY_CONFIG, default_return=TavilyResponse(query="搜索失败"))
def _search_internal(self, **kwargs) -> TavilyResponse:
    """Internal search executor với retry mechanism"""
    try:
        kwargs['topic'] = 'general'
        api_params = {k: v for k, v in kwargs.items() if v is not None}
        response_dict = self._client.search(**api_params)
        
        # Process response...
        return TavilyResponse(...)
        
    except Exception as e:
        print(f"搜索时发生错误: {str(e)}")
        raise e  # Let retry mechanism handle it
```

## Cấu hình và Tùy chỉnh

### Settings Structure

```python
class Settings(BaseSettings):
    # Tavily Configuration
    TAVILY_API_KEY: str
    
    # Search Configuration
    DEFAULT_MAX_RESULTS: int = 7
    SEARCH_CONTENT_MAX_LENGTH: int = 10000
    
    # Processing Configuration
    MAX_REFLECTIONS: int = 2
    
    # Output Configuration
    OUTPUT_DIR: str = "query_engine_streamlit_reports"
    SAVE_INTERMEDIATE_STATES: bool = True
```

### Tùy chỉnh Search Tools

```python
# Tùy chỉnh các tham số search
search_config = {
    'basic_search_news': {
        'max_results': 10,
        'search_depth': 'basic'
    },
    'deep_search_news': {
        'max_results': 25,
        'search_depth': 'advanced',
        'include_answer': 'advanced'
    },
    'search_news_by_date': {
        'max_results': 20,
        'date_validation': True
    }
}
```

## Hiệu suất và Tối ưu

### 1. API Rate Limiting

```python
import time
from typing import Dict

class TavilyRateLimiter:
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

class NewsSearchCache:
    def __init__(self, max_size: int = 1000):
        self.cache = {}
        self.max_size = max_size
    
    def get_cache_key(self, query: str, tool: str, **kwargs) -> str:
        key_data = f"{query}:{tool}:{sorted(kwargs.items())}"
        return hashlib.md5(key_data.encode()).hexdigest()
    
    def get(self, cache_key: str) -> Optional[TavilyResponse]:
        return self.cache.get(cache_key)
    
    def set(self, cache_key: str, response: TavilyResponse):
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

def parallel_news_search(self, queries: List[str], tool_name: str) -> List[TavilyResponse]:
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
                logger.exception(f"Parallel news search failed: {e}")
        
        return results
```

## Tích hợp với Hệ thống

### 1. Streamlit Integration

```python
# SingleEngineApp/query_engine_streamlit_app.py
import streamlit as st
from QueryEngine import create_agent

@st.cache_resource
def get_agent():
    return create_agent()

def main():
    agent = get_agent()
    
    st.title("QueryEngine - Phân tích Tin tức")
    
    query = st.text_input("Nhập truy vấn tin tức:")
    
    if st.button("Bắt đầu nghiên cứu") and query:
        with st.spinner("Đang phân tích tin tức..."):
            report = agent.research(query, save_report=True)
        
        st.markdown(report)
        
        # Hiển thị thông tin tin tức
        if hasattr(agent.state, 'news_sources'):
            st.json(agent.state.news_sources)
```

### 2. ForumEngine Integration

QueryEngine ghi log structured để ForumEngine có thể xử lý:

```python
# Log format cho ForumEngine
logger.info(f"[FirstSummaryNode] 正在生成首次段落总结")
logger.info(f"清理后的输出: {json.dumps(news_summary_data)}")

# Log thông tin tin tức
if search_response.results:
    logger.info(f"Found {len(search_response.results)} news articles")
    
    # Log ngày phát hành
    for result in search_response.results:
        if result.published_date:
            logger.info(f"News published on: {result.published_date}")
```

### 3. ReportEngine Integration

QueryEngine cung cấp báo cáo tin tức cho ReportEngine:

```python
# ReportEngine đọc báo cáo từ QueryEngine
def load_input_files(self, file_paths: Dict[str, str]):
    with open(file_paths['query'], 'r', encoding='utf-8') as f:
        query_report = f.read()
    content['reports'].append(query_report)
    
    # Đọc thêm metadata tin tức nếu có
    news_metadata_file = file_paths.get('query_metadata')
    if news_metadata_file and os.path.exists(news_metadata_file):
        with open(news_metadata_file, 'r', encoding='utf-8') as f:
            content['news_metadata'] = json.load(f)
```

## Troubleshooting và Debugging

### Các Vấn đề Thường gặp

1. **Tavily API Connection**: Kiểm tra API key và rate limits
2. **Date Format Validation**: Xác thực format YYYY-MM-DD
3. **News Source Credibility**: Đánh giá độ tin cậy nguồn tin
4. **Breaking News Detection**: Debug phát hiện tin tức nóng

### Debug Tools

```python
# Debug mode cho Tavily search
if self.config.DEBUG_MODE:
    logger.info(f"Debug: search_query={search_query}")
    logger.info(f"Debug: search_tool={search_tool}")
    logger.info(f"Debug: response={response_dict}")

# News content inspection
def inspect_news_content(self, search_response: TavilyResponse):
    return {
        "articles_count": len(search_response.results),
        "images_count": len(search_response.images),
        "has_ai_summary": bool(search_response.answer),
        "response_time": search_response.response_time,
        "publication_dates": [r.published_date for r in search_response.results if r.published_date],
        "source_domains": list(set([urlparse(r.url).netloc for r in search_response.results]))
    }
```

## Kết luận

QueryEngine là một động cơ nghiên cứu tin tức mạnh mẽ, kết hợp:

- **6 công cụ tìm kiếm chuyên dụng** từ Tavily News
- **Khả năng phân tích thời sự** với breaking news detection
- **Fact checking và verification** để đảm bảo độ tin cậy
- **Multi-source analysis** để có cái nhìn toàn diện
- **Date-based search** cho nghiên cứu lịch sử tin tức

Với kiến trúc được thiết kế đặc biệt cho tin tức thời sự, QueryEngine cung cấp khả năng phân tích tin tức chuyên nghiệp, là một trong ba trụ cột chính của hệ thống BettaFish, chuyên biệt trong việc xử lý và phân tích thông tin tin tức từ các nguồn toàn cầu.