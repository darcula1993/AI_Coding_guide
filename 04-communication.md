# AI Coding 指南 - 第四章：与AI有效沟通的技巧

## 核心沟通原则

### 1. 把AI当作"人"来沟通

AI需要上下文、需要明确的指令、需要反馈。就像与一个有能力但需要指导的初级开发者合作一样。

### 2. 渐进式对话，不要一次性提出复杂需求

**❌ 错误的沟通方式：**
```
Human: 帮我实现一个完整的电商系统，包括用户管理、商品管理、订单处理、支付集成、库存管理、优惠券系统、评价系统等所有功能。
```

**✅ 正确的沟通方式：**
```
Human: 我要开发一个电商系统。我们先从用户管理模块开始，首先实现用户注册和登录功能。项目使用Django，数据库是PostgreSQL。

AI: 好的，我来帮你实现用户注册和登录功能。首先让我确认几个细节：

1. 用户登录是使用邮箱还是用户名？
2. 是否需要邮箱验证？
3. 密码有什么具体要求？
4. 是否需要记住登录状态？
```

## 精确描述问题的技巧

### 3. 提供丰富且准确的上下文信息

#### 描述现状 vs 期望状态

```
❌ 模糊的描述：
Human: 我的代码有问题，帮我修复一下。

✅ 清晰的描述：
Human: 我的用户注册API出现了问题。

**现状：**
- 用户提交注册表单后，API返回500错误
- 错误日志显示：IntegrityError: duplicate key value violates unique constraint "users_email_key"
- 但是我确认数据库中没有重复的邮箱

**期望状态：**
- 用户注册成功时返回201状态码和用户信息
- 邮箱重复时返回400状态码和明确的错误信息
- 其他验证错误也要有相应的错误提示

**相关代码：**
```python
@api_view(['POST'])
def register_user(request):
    serializer = UserSerializer(data=request.data)
    if serializer.is_valid():
        user = serializer.save()
        return Response({'user_id': user.id}, status=201)
    return Response(serializer.errors, status=400)
```

**技术栈：**
- Django REST Framework 3.14
- PostgreSQL 14
- Python 3.9
```

### 4. 约束AI的修改范围

#### 明确修改边界

```
Human: 我需要修复用户注册功能中的邮箱验证问题。

**修改范围限制：**
- 只修改 user_views.py 中的 register_user 函数
- 不要修改 User 模型
- 不要影响现有的登录功能
- 保持现有的API接口格式不变

**具体问题：**
邮箱格式验证不够严格，需要添加更完善的验证规则。

AI: 明白了，我只会修改register_user函数中的邮箱验证逻辑，不会触及其他部分。让我为你实现更严格的邮箱验证：

```python
import re
from django.core.exceptions import ValidationError

@api_view(['POST'])
def register_user(request):
    email = request.data.get('email', '')
    
    # 添加严格的邮箱验证
    email_pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    if not re.match(email_pattern, email):
        return Response(
            {'error': 'Invalid email format'}, 
            status=400
        )
    
    # 检查邮箱是否已存在
    if User.objects.filter(email=email).exists():
        return Response(
            {'error': 'Email already registered'}, 
            status=400
        )
    
    serializer = UserSerializer(data=request.data)
    if serializer.is_valid():
        user = serializer.save()
        return Response({'user_id': user.id}, status=201)
    return Response(serializer.errors, status=400)
```

这样修改只涉及指定的函数，不会影响其他功能。
```

## 问题定位的沟通策略

### 5. 当不清楚问题根源时的对话流程

#### 分析式对话模式

```
Human: 我的应用在生产环境偶尔出现500错误，但本地开发环境一切正常。你能帮我分析可能的原因吗？

AI: 生产环境500错误且本地正常的情况比较常见。让我帮你系统性地分析可能的原因：

**1. 数据差异问题**
- 生产数据量是否比本地大很多？
- 是否有特殊的数据格式在生产环境中存在？

**2. 环境配置差异**
- 数据库版本是否一致？
- Python包版本是否一致？
- 环境变量配置是否正确？

**3. 资源限制问题**
- 内存使用情况如何？
- 数据库连接池是否够用？
- 请求并发量是否过高？

**4. 网络和依赖服务**
- 是否依赖外部API服务？
- 网络延迟是否影响超时设置？

为了进一步定位问题，你能告诉我：
1. 错误是否有特定的触发模式？
2. 服务器日志中的具体错误信息是什么？
3. 应用的主要技术栈是什么？
```

## 技术规格说明的最佳实践

### 6. 指定技术栈和依赖

#### 明确技术选择

```
Human: 帮我实现一个缓存系统。

**项目技术栈：**
- 后端：Python 3.9 + FastAPI
- 缓存：Redis 6.2
- 数据库：PostgreSQL 13
- 容器化：Docker

**功能需求：**
- 支持键值对存储
- 支持过期时间设置
- 提供缓存统计信息
- 支持批量操作

**性能要求：**
- 单次查询响应时间 < 10ms
- 支持10000+ QPS
- 内存使用效率优化

AI: 基于你的技术栈，我来设计一个高性能的缓存系统。使用Redis作为缓存后端，FastAPI提供API接口：

```python
# cache_service.py
import redis
import json
import time
from typing import Any, Dict, List, Optional, Union
from datetime import timedelta
from fastapi import HTTPException

class CacheService:
    """高性能缓存服务"""
    
    def __init__(self, redis_host: str = "localhost", redis_port: int = 6379, 
                 redis_db: int = 0, redis_password: Optional[str] = None):
        """
        初始化缓存服务
        
        Args:
            redis_host: Redis主机地址
            redis_port: Redis端口
            redis_db: Redis数据库编号
            redis_password: Redis密码
        """
        self.redis_client = redis.Redis(
            host=redis_host,
            port=redis_port,
            db=redis_db,
            password=redis_password,
            decode_responses=True,
            socket_keepalive=True,
            socket_keepalive_options={},
            health_check_interval=30
        )
        
        # 统计信息
        self.stats = {
            'hits': 0,
            'misses': 0,
            'sets': 0,
            'deletes': 0
        }
    
    def get(self, key: str) -> Optional[Any]:
        """获取缓存值"""
        try:
            value = self.redis_client.get(key)
            if value is not None:
                self.stats['hits'] += 1
                return json.loads(value)
            else:
                self.stats['misses'] += 1
                return None
        except Exception as e:
            raise HTTPException(status_code=500, detail=f"Cache get error: {e}")
    
    def set(self, key: str, value: Any, expire_seconds: Optional[int] = None) -> bool:
        """设置缓存值"""
        try:
            json_value = json.dumps(value, ensure_ascii=False)
            result = self.redis_client.set(key, json_value, ex=expire_seconds)
            if result:
                self.stats['sets'] += 1
            return result
        except Exception as e:
            raise HTTPException(status_code=500, detail=f"Cache set error: {e}")
    
    def mget(self, keys: List[str]) -> Dict[str, Any]:
        """批量获取缓存值"""
        try:
            values = self.redis_client.mget(keys)
            result = {}
            
            for i, key in enumerate(keys):
                if values[i] is not None:
                    result[key] = json.loads(values[i])
                    self.stats['hits'] += 1
                else:
                    self.stats['misses'] += 1
            
            return result
        except Exception as e:
            raise HTTPException(status_code=500, detail=f"Cache mget error: {e}")
    
    def mset(self, mapping: Dict[str, Any], expire_seconds: Optional[int] = None) -> bool:
        """批量设置缓存值"""
        try:
            # 序列化所有值
            serialized_mapping = {
                key: json.dumps(value, ensure_ascii=False) 
                for key, value in mapping.items()
            }
            
            pipe = self.redis_client.pipeline()
            pipe.mset(serialized_mapping)
            
            # 如果设置了过期时间，为每个key设置过期时间
            if expire_seconds:
                for key in mapping.keys():
                    pipe.expire(key, expire_seconds)
            
            results = pipe.execute()
            
            if results[0]:  # mset的结果
                self.stats['sets'] += len(mapping)
            
            return results[0]
        except Exception as e:
            raise HTTPException(status_code=500, detail=f"Cache mset error: {e}")
    
    def delete(self, key: str) -> bool:
        """删除缓存值"""
        try:
            result = self.redis_client.delete(key)
            if result:
                self.stats['deletes'] += 1
            return bool(result)
        except Exception as e:
            raise HTTPException(status_code=500, detail=f"Cache delete error: {e}")
    
    def get_stats(self) -> Dict[str, Union[int, float]]:
        """获取缓存统计信息"""
        total_requests = self.stats['hits'] + self.stats['misses']
        hit_rate = self.stats['hits'] / total_requests if total_requests > 0 else 0
        
        # 获取Redis内存使用信息
        info = self.redis_client.info('memory')
        
        return {
            'hits': self.stats['hits'],
            'misses': self.stats['misses'],
            'sets': self.stats['sets'],
            'deletes': self.stats['deletes'],
            'hit_rate': round(hit_rate, 4),
            'memory_used': info.get('used_memory', 0),
            'memory_used_human': info.get('used_memory_human', '0B')
        }

# FastAPI路由
from fastapi import FastAPI, Depends

app = FastAPI()
cache_service = CacheService()

@app.get("/cache/{key}")
async def get_cache(key: str):
    """获取缓存"""
    value = cache_service.get(key)
    if value is None:
        raise HTTPException(status_code=404, detail="Key not found")
    return {"key": key, "value": value}

@app.post("/cache/{key}")
async def set_cache(key: str, request: dict):
    """设置缓存"""
    value = request.get("value")
    expire_seconds = request.get("expire_seconds")
    
    result = cache_service.set(key, value, expire_seconds)
    return {"success": result}

@app.get("/cache/stats")
async def get_cache_stats():
    """获取缓存统计"""
    return cache_service.get_stats()
```

**这个实现的特点：**
1. ✅ 使用连接池提高性能
2. ✅ 支持批量操作减少网络往返  
3. ✅ 内置统计信息监控
4. ✅ 完整的错误处理
5. ✅ 符合你的FastAPI技术栈
```

## 沟通效率优化技巧

### 7. 使用共同语言

建立一套标准的术语和表达方式，让AI能够准确理解你的意图：

```python
# 建立项目词汇表
PROJECT_TERMS = {
    "用户": "User model with authentication",
    "商品": "Product model with inventory tracking", 
    "订单": "Order model with status workflow",
    "支付": "Payment integration with third-party providers",
    "库存": "Inventory management with real-time tracking",
    "优惠券": "Coupon system with validation rules"
}

# 在对话中使用一致的术语
Human: 我需要在订单模块中添加优惠券功能，当用户下单时能够应用优惠券折扣。

# 而不是混合使用不同的词汇
Human: 我需要在order里面加个coupon的东西，用户买东西的时候可以打折。
```

### 8. 监控AI的思考过程

观察AI的推理和决策过程，及时纠正偏差：

```
Human: 帮我优化这个数据库查询的性能。

AI: 我注意到这个查询可能存在性能问题。让我分析一下：

1. 查询涉及多个表的连接
2. 没有使用适当的索引
3. 可能存在N+1查询问题

我的优化策略是：
1. 添加复合索引
2. 使用查询优化技术
3. 实现结果缓存

你觉得这个分析方向对吗？还是你有其他的性能瓶颈需要重点关注？
```

## 总结：高效AI沟通检查清单

### ✅ 开始对话前

- [ ] 明确项目的技术栈和依赖
- [ ] 准备好相关的代码和错误信息
- [ ] 确定修改的范围和边界
- [ ] 制定分步骤的实施计划

### ✅ 对话过程中

- [ ] 使用结构化的问题描述
- [ ] 提供充分的上下文信息
- [ ] 监控AI的理解和推理过程
- [ ] 及时纠正AI的偏差和误解
- [ ] 保持术语的一致性

### ✅ 代码生成后

- [ ] 仔细审查AI生成的代码
- [ ] 验证代码的正确性和安全性
- [ ] 确认修改范围符合预期
- [ ] 进行必要的测试和验证

通过掌握这些沟通技巧，你可以显著提高AI Coding的效率和质量，建立更流畅的人机协作关系。