1. 装饰器本质
装饰器用于在不修改原函数核心逻辑的前提下，为函数增加功能。

from collections.abc import Callable
from functools import wraps
from typing import Any

def log_call(func: Callable[..., Any]) -> Callable[..., Any]:
    @wraps(func)
    def wrapper(*args: Any, **kwargs: Any) -> Any:
        print(f"Calling: {func.__name__}")
        result = func(*args, **kwargs)
        print(f"Finished: {func.__name__}")
        return result

    return wrapper


@log_call
def add(a: int, b: int) -> int:
    return a + b


print(add(3, 5))


2. 带参数的装饰器

from collections.abc import Callable
from functools import wraps
from typing import Any

def retry(times: int):
    def decorator(func: Callable[..., Any]) -> Callable[..., Any]:
        @wraps(func)
        def wrapper(*args: Any, **kwargs: Any) -> Any:
            last_error = None

            for attempt in range(1, times + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as exc:
                    last_error = exc
                    print(f"Attempt {attempt}/{times} failed: {exc}")

            raise last_error

        return wrapper

    return decorator
	
	
@retry(times=3)
def unstable_task():
    raise RuntimeError("Network error")


3. 常见装饰器场景
	日志记录
	权限检查
	缓存
	重试机制
	性能统计
	限流
	事务控制
	Web 路由注册

泛型

from typing import TypeVar

T = TypeVar("T")

def first_item(items: list[T]) -> T:
    if not items:
        raise ValueError("items cannot be empty")
    return items[0]
	
	
	
dataclass：≈ Java DTO/VO/POJO（轻量、默认不校验）
TypedDict：≈ “有 schema 约束的 dict 类型提示”（只给静态检查看，不负责运行时）
Pydantic：≈ Java DTO/VO + 校验 + JSON(反)序列化能力（边界层/协议层利器）



操作系统进程
└── Python 线程
    └── asyncio 事件循环
        ├── 协程 / Task A
        ├── 协程 / Task B
        ├── 协程 / Task C
        └── ...
		

一个 Python 进程
├── 全局变量、堆内存、模块对象
├── HTTP 连接池
├── Redis 连接池
├── 数据库连接池
│
└── 一个线程
    ├── 线程栈
    ├── thread-local 数据
    └── asyncio 事件循环
        ├── 协程 A
        ├── 协程 B
        └── 协程 C


		

对比项	线程	协程
调度者	操作系统	Python 事件循环
切换成本	相对较高	很低
内存占用	相对较多	相对较少
是否天然并行	可由 OS 多核调度，但受 GIL 限制	单线程事件循环通常不并行
适合场景	阻塞式 I/O、旧同步库	大量异步 I/O

