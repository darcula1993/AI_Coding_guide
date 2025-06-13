# AI Coding 指南 - 第九章：编码规范与最佳实践

## AI Coding 编码规范

### 1. 代码质量标准

```python
# coding_standards.py
"""AI Coding 团队编码标准"""

from typing import Dict, List, Optional, Union, Any
from dataclasses import dataclass
from abc import ABC, abstractmethod
import logging

# ✅ 好的实践：完整的类型注解
@dataclass
class UserProfile:
    """用户资料数据类"""
    user_id: int
    email: str
    name: str
    age: Optional[int] = None
    is_active: bool = True
    
    def to_dict(self) -> Dict[str, Any]:
        """转换为字典格式"""
        return {
            'user_id': self.user_id,
            'email': self.email,
            'name': self.name,
            'age': self.age,
            'is_active': self.is_active
        }

# ✅ 好的实践：清晰的接口定义
class UserServiceInterface(ABC):
    """用户服务接口"""
    
    @abstractmethod
    def create_user(self, user_data: Dict[str, Any]) -> UserProfile:
        """创建用户"""
        pass
    
    @abstractmethod
    def get_user_by_id(self, user_id: int) -> Optional[UserProfile]:
        """根据ID获取用户"""
        pass

# ✅ 好的实践：完整的错误处理
class UserService(UserServiceInterface):
    """用户服务实现"""
    
    def __init__(self, user_repository: 'UserRepository'):
        self.user_repository = user_repository
        self.logger = logging.getLogger(__name__)
    
    def create_user(self, user_data: Dict[str, Any]) -> UserProfile:
        """
        创建新用户
        
        Args:
            user_data: 包含用户信息的字典
            
        Returns:
            UserProfile: 创建的用户资料
            
        Raises:
            ValidationError: 输入数据验证失败
            DuplicateUserError: 用户已存在
            DatabaseError: 数据库操作失败
        """
        try:
            # 输入验证
            self._validate_user_data(user_data)
            
            # 业务逻辑检查
            if self._user_exists(user_data['email']):
                raise DuplicateUserError(f"User with email {user_data['email']} already exists")
            
            # 执行创建
            user_profile = self.user_repository.create(user_data)
            
            # 记录日志
            self.logger.info(f"Created user: {user_profile.user_id}")
            
            return user_profile
            
        except ValidationError as e:
            self.logger.warning(f"User creation validation failed: {e}")
            raise
        except Exception as e:
            self.logger.error(f"Unexpected error creating user: {e}")
            raise DatabaseError("Failed to create user") from e
    
    def _validate_user_data(self, user_data: Dict[str, Any]) -> None:
        """验证用户数据"""
        required_fields = ['email', 'name']
        
        for field in required_fields:
            if field not in user_data or not user_data[field]:
                raise ValidationError(f"Missing required field: {field}")
        
        # 邮箱格式验证
        if not self._is_valid_email(user_data['email']):
            raise ValidationError("Invalid email format")
    
    def _is_valid_email(self, email: str) -> bool:
        """验证邮箱格式"""
        import re
        pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        return bool(re.match(pattern, email))
    
    def _user_exists(self, email: str) -> bool:
        """检查用户是否存在"""
        return self.user_repository.get_by_email(email) is not None

# ❌ 不好的实践示例（避免这样写）
class BadUserService:
    """不好的用户服务实现示例"""
    
    def create(self, data):  # ❌ 缺少类型注解
        # ❌ 缺少文档字符串
        # ❌ 缺少输入验证
        # ❌ 缺少错误处理
        # ❌ 变量名不清晰
        u = User()
        u.e = data['email']
        u.n = data['name']
        u.save()  # ❌ 没有处理可能的异常
        return u
```

### 2. 命名规范

```python
# naming_conventions.py
"""命名规范示例"""

# ✅ 好的命名实践
class OrderManagementService:
    """订单管理服务"""
    
    def __init__(self, order_repository: 'OrderRepository', 
                 payment_service: 'PaymentService'):
        self.order_repository = order_repository
        self.payment_service = payment_service
        self.logger = logging.getLogger(__name__)
    
    def create_new_order(self, customer_id: int, order_items: List[Dict]) -> Dict:
        """创建新订单"""
        total_amount = self._calculate_total_amount(order_items)
        
        order_data = {
            'customer_id': customer_id,
            'items': order_items,
            'total_amount': total_amount,
            'status': OrderStatus.PENDING,
            'created_at': datetime.utcnow()
        }
        
        return self.order_repository.create_order(order_data)
    
    def process_payment_for_order(self, order_id: int, payment_method: str) -> bool:
        """处理订单支付"""
        order = self.order_repository.get_order_by_id(order_id)
        
        if not order:
            raise OrderNotFoundError(f"Order {order_id} not found")
        
        payment_result = self.payment_service.process_payment(
            amount=order['total_amount'],
            payment_method=payment_method
        )
        
        if payment_result.is_successful:
            self._update_order_status(order_id, OrderStatus.PAID)
            return True
        
        return False
    
    def _calculate_total_amount(self, order_items: List[Dict]) -> float:
        """计算订单总金额"""
        return sum(
            item['price'] * item['quantity'] 
            for item in order_items
        )
    
    def _update_order_status(self, order_id: int, new_status: OrderStatus) -> None:
        """更新订单状态"""
        self.order_repository.update_order_status(order_id, new_status)
        self.logger.info(f"Order {order_id} status updated to {new_status}")

# ❌ 不好的命名示例（避免这样写）
class OMS:  # ❌ 缩写不清晰
    def __init__(self, r, p):  # ❌ 参数名不清晰
        self.r = r  # ❌ 变量名无意义
        self.p = p
    
    def create(self, c, i):  # ❌ 函数名和参数名都不清晰
        t = 0  # ❌ 变量名无意义
        for x in i:  # ❌ 循环变量名不清晰
            t += x['price'] * x['qty']  # ❌ 字段名不一致
        
        return self.r.create({
            'c': c,  # ❌ 字段名无意义
            'i': i,
            't': t
        })
```

### 3. 安全编程实践

```python
# security_practices.py
"""安全编程实践"""

import hashlib
import secrets
import re
from typing import Dict, Any, Optional
import logging

class SecureUserService:
    """安全的用户服务实现"""
    
    def __init__(self):
        self.logger = logging.getLogger(__name__)
        # ✅ 使用安全的随机数生成器
        self.token_generator = secrets.SystemRandom()
    
    def create_user_securely(self, user_data: Dict[str, Any]) -> Dict[str, Any]:
        """安全地创建用户"""
        
        # ✅ 输入验证和清理
        sanitized_data = self._sanitize_user_input(user_data)
        
        # ✅ 密码安全处理
        if 'password' in sanitized_data:
            sanitized_data['password_hash'] = self._hash_password(
                sanitized_data.pop('password')
            )
        
        # ✅ 生成安全的用户ID
        user_id = self._generate_secure_user_id()
        sanitized_data['user_id'] = user_id
        
        # ✅ 记录安全审计日志（不包含敏感信息）
        self.logger.info(f"User creation attempt for email: {sanitized_data.get('email', 'UNKNOWN')}")
        
        return sanitized_data
    
    def _sanitize_user_input(self, user_data: Dict[str, Any]) -> Dict[str, Any]:
        """清理用户输入"""
        sanitized = {}
        
        # ✅ 邮箱验证和清理
        if 'email' in user_data:
            email = str(user_data['email']).strip().lower()
            if self._is_valid_email(email):
                sanitized['email'] = email
            else:
                raise ValidationError("Invalid email format")
        
        # ✅ 用户名清理（移除潜在的脚本注入）
        if 'name' in user_data:
            name = str(user_data['name']).strip()
            # 移除HTML标签和特殊字符
            name = re.sub(r'<[^>]+>', '', name)
            name = re.sub(r'[<>&"\'`]', '', name)
            
            if 2 <= len(name) <= 50:
                sanitized['name'] = name
            else:
                raise ValidationError("Name must be between 2 and 50 characters")
        
        # ✅ 密码强度验证
        if 'password' in user_data:
            password = str(user_data['password'])
            if self._is_strong_password(password):
                sanitized['password'] = password
            else:
                raise ValidationError("Password does not meet security requirements")
        
        return sanitized
    
    def _hash_password(self, password: str) -> str:
        """安全地哈希密码"""
        # ✅ 使用bcrypt或类似的安全哈希算法
        import bcrypt
        salt = bcrypt.gensalt()
        return bcrypt.hashpw(password.encode('utf-8'), salt).decode('utf-8')
    
    def _is_strong_password(self, password: str) -> bool:
        """检查密码强度"""
        if len(password) < 8:
            return False
        
        # 检查是否包含大小写字母、数字和特殊字符
        has_upper = bool(re.search(r'[A-Z]', password))
        has_lower = bool(re.search(r'[a-z]', password))
        has_digit = bool(re.search(r'\d', password))
        has_special = bool(re.search(r'[!@#$%^&*(),.?":{}|<>]', password))
        
        return all([has_upper, has_lower, has_digit, has_special])
    
    def _generate_secure_user_id(self) -> str:
        """生成安全的用户ID"""
        # ✅ 使用加密安全的随机数生成器
        return secrets.token_hex(16)
    
    def _is_valid_email(self, email: str) -> bool:
        """验证邮箱格式"""
        pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        return bool(re.match(pattern, email)) and len(email) <= 254

# ✅ 安全的数据库查询实践
class SecureDatabaseService:
    """安全的数据库服务"""
    
    def get_user_by_email(self, email: str) -> Optional[Dict]:
        """安全地通过邮箱查询用户"""
        
        # ✅ 使用参数化查询防止SQL注入
        query = "SELECT user_id, email, name FROM users WHERE email = %s"
        
        try:
            # ✅ 不要记录敏感信息到日志
            self.logger.info("Executing user lookup query")
            
            result = self.db_connection.execute(query, (email,))
            return result.fetchone()
            
        except Exception as e:
            # ✅ 记录错误但不暴露内部信息
            self.logger.error("Database query failed", exc_info=True)
            raise DatabaseError("Query execution failed")
    
    def update_user_profile(self, user_id: str, profile_data: Dict[str, Any]) -> bool:
        """安全地更新用户资料"""
        
        # ✅ 白名单验证允许更新的字段
        allowed_fields = {'name', 'age', 'phone'}
        update_fields = {k: v for k, v in profile_data.items() if k in allowed_fields}
        
        if not update_fields:
            raise ValidationError("No valid fields to update")
        
        # ✅ 动态构建安全的更新查询
        set_clause = ', '.join(f"{field} = %s" for field in update_fields.keys())
        query = f"UPDATE users SET {set_clause} WHERE user_id = %s"
        
        values = list(update_fields.values()) + [user_id]
        
        try:
            self.db_connection.execute(query, values)
            return True
        except Exception as e:
            self.logger.error(f"Failed to update user profile: {e}")
            return False

# ❌ 不安全的实践示例（绝对避免）
class InsecureService:
    """不安全的服务实现示例（请勿使用）"""
    
    def bad_user_query(self, email):
        # ❌ SQL注入漏洞
        query = f"SELECT * FROM users WHERE email = '{email}'"
        return self.db.execute(query)
    
    def bad_password_storage(self, password):
        # ❌ 明文存储密码
        return password
    
    def bad_logging(self, user_data):
        # ❌ 记录敏感信息到日志
        self.logger.info(f"User data: {user_data}")  # 包含密码等敏感信息
```

### 4. 性能优化实践

```python
# performance_practices.py
"""性能优化最佳实践"""

import asyncio
import functools
import time
from typing import List, Dict, Any, Optional
from concurrent.futures import ThreadPoolExecutor
import caching

class PerformantUserService:
    """高性能用户服务"""
    
    def __init__(self):
        self.cache = caching.Cache()
        self.thread_pool = ThreadPoolExecutor(max_workers=10)
    
    @functools.lru_cache(maxsize=128)
    def get_user_permissions(self, user_id: int) -> List[str]:
        """获取用户权限（带缓存）"""
        # ✅ 使用缓存减少数据库查询
        cache_key = f"user_permissions:{user_id}"
        cached_result = self.cache.get(cache_key)
        
        if cached_result:
            return cached_result
        
        permissions = self._query_user_permissions_from_db(user_id)
        
        # ✅ 缓存结果，设置过期时间
        self.cache.set(cache_key, permissions, ttl=300)  # 5分钟过期
        
        return permissions
    
    async def batch_get_users(self, user_ids: List[int]) -> Dict[int, Dict]:
        """批量获取用户信息"""
        # ✅ 批量操作减少数据库往返
        if not user_ids:
            return {}
        
        # 检查缓存
        cached_users = {}
        uncached_ids = []
        
        for user_id in user_ids:
            cache_key = f"user:{user_id}"
            cached_user = self.cache.get(cache_key)
            if cached_user:
                cached_users[user_id] = cached_user
            else:
                uncached_ids.append(user_id)
        
        # 批量查询未缓存的用户
        if uncached_ids:
            db_users = await self._batch_query_users(uncached_ids)
            
            # 更新缓存
            for user_id, user_data in db_users.items():
                cache_key = f"user:{user_id}"
                self.cache.set(cache_key, user_data, ttl=600)  # 10分钟过期
            
            cached_users.update(db_users)
        
        return cached_users
    
    async def _batch_query_users(self, user_ids: List[int]) -> Dict[int, Dict]:
        """批量查询用户数据"""
        # ✅ 使用IN查询替代多个单独查询
        placeholders = ','.join(['%s'] * len(user_ids))
        query = f"SELECT user_id, email, name FROM users WHERE user_id IN ({placeholders})"
        
        results = await self.db.fetch_all(query, user_ids)
        
        return {
            row['user_id']: dict(row)
            for row in results
        }
    
    async def process_users_concurrently(self, user_ids: List[int]) -> List[Dict]:
        """并发处理用户数据"""
        # ✅ 使用异步并发提高性能
        async def process_single_user(user_id: int) -> Dict:
            user_data = await self.get_user_by_id(user_id)
            permissions = await self.get_user_permissions_async(user_id)
            
            return {
                **user_data,
                'permissions': permissions
            }
        
        # ✅ 控制并发数量避免资源耗尽
        semaphore = asyncio.Semaphore(20)  # 最多20个并发
        
        async def process_with_semaphore(user_id: int):
            async with semaphore:
                return await process_single_user(user_id)
        
        tasks = [process_with_semaphore(user_id) for user_id in user_ids]
        results = await asyncio.gather(*tasks, return_exceptions=True)
        
        # ✅ 处理异常结果
        successful_results = [
            result for result in results 
            if not isinstance(result, Exception)
        ]
        
        return successful_results
    
    def optimize_database_query(self, filters: Dict[str, Any]) -> List[Dict]:
        """优化数据库查询"""
        # ✅ 构建高效的查询
        base_query = "SELECT user_id, email, name FROM users"
        where_conditions = []
        query_params = []
        
        # ✅ 动态构建WHERE子句
        if filters.get('email'):
            where_conditions.append("email = %s")
            query_params.append(filters['email'])
        
        if filters.get('age_min'):
            where_conditions.append("age >= %s")
            query_params.append(filters['age_min'])
        
        if filters.get('is_active') is not None:
            where_conditions.append("is_active = %s")
            query_params.append(filters['is_active'])
        
        if where_conditions:
            query = f"{base_query} WHERE {' AND '.join(where_conditions)}"
        else:
            query = base_query
        
        # ✅ 添加索引提示（如果数据库支持）
        query += " ORDER BY user_id"
        
        # ✅ 限制结果数量
        if filters.get('limit'):
            query += f" LIMIT {int(filters['limit'])}"
        
        return self.db.fetch_all(query, query_params)

# ❌ 性能不佳的实践示例（避免这样写）
class SlowUserService:
    """性能不佳的用户服务示例"""
    
    def get_users_slowly(self, user_ids: List[int]) -> List[Dict]:
        # ❌ N+1查询问题
        users = []
        for user_id in user_ids:
            user = self.db.execute("SELECT * FROM users WHERE user_id = %s", (user_id,))
            if user:
                # ❌ 每个用户都单独查询权限
                permissions = self.db.execute(
                    "SELECT permission FROM user_permissions WHERE user_id = %s", 
                    (user_id,)
                )
                user['permissions'] = permissions
                users.append(user)
        return users
    
    def no_caching_example(self, user_id: int) -> Dict:
        # ❌ 没有缓存，每次都查询数据库
        return self.db.execute("SELECT * FROM users WHERE user_id = %s", (user_id,))
```

## 最佳实践总结

### ✅ AI Coding 最佳实践检查清单

**代码质量：**
- [ ] 完整的类型注解
- [ ] 清晰的命名规范
- [ ] 详细的文档字符串
- [ ] 完善的错误处理
- [ ] 适当的日志记录

**安全性：**
- [ ] 输入验证和清理
- [ ] 参数化查询防止注入
- [ ] 安全的密码处理
- [ ] 敏感信息保护
- [ ] 访问控制检查

**性能：**
- [ ] 适当的缓存策略
- [ ] 批量操作优化
- [ ] 异步并发处理
- [ ] 数据库查询优化
- [ ] 资源使用控制

**可维护性：**
- [ ] 模块化设计
- [ ] 单一职责原则
- [ ] 依赖注入
- [ ] 接口抽象
- [ ] 测试覆盖

遵循这些最佳实践，可以确保AI生成的代码质量高、安全性强、性能优越且易于维护。