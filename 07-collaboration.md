# AI Coding 指南 - 第七章：团队协作与工具选择

## 团队AI Coding的挑战

多人使用AI编程时容易出现：
- 代码风格不一致
- AI生成的代码冲突
- 技术选择分歧
- 沟通成本增加

## 团队协作最佳实践

### 1. 统一AI工具选择

**团队标准化配置：**

```python
# team_ai_config.py
"""团队AI编程配置标准"""

TEAM_AI_STANDARDS = {
    "primary_ai_tool": "Claude/GPT-4",  # 主要AI工具
    "code_style": "black + flake8",     # 代码格式化
    "testing_framework": "pytest",      # 测试框架
    "documentation": "sphinx",          # 文档工具
    "type_checking": "mypy",            # 类型检查
}

# AI提示词模板
PROMPT_TEMPLATES = {
    "feature_development": """
    请实现{feature_name}功能：
    
    **项目背景：**
    - 技术栈：{tech_stack}
    - 编码规范：遵循团队{code_style}标准
    - 测试要求：使用{testing_framework}编写测试
    
    **功能需求：**
    {requirements}
    
    **约束条件：**
    - 只修改{allowed_files}
    - 保持与{existing_modules}的兼容性
    - 遵循{architectural_pattern}架构模式
    """,
    
    "bug_fix": """
    请帮我修复以下bug：
    
    **问题描述：**{bug_description}
    **错误信息：**{error_message}
    **影响范围：**{scope}
    **修复约束：**不要影响{protected_modules}
    """
}

def get_prompt_template(template_type: str, **kwargs) -> str:
    """获取标准化的AI提示词"""
    template = PROMPT_TEMPLATES.get(template_type)
    if not template:
        raise ValueError(f"Unknown template type: {template_type}")
    
    return template.format(**kwargs)
```

### 2. 代码风格统一

```python
# .pre-commit-config.yaml - 团队代码质量检查
repos:
  - repo: https://github.com/psf/black
    rev: 23.1.0
    hooks:
      - id: black
        language_version: python3.9
        args: [--line-length=88]

  - repo: https://github.com/pycqa/isort
    rev: 5.12.0
    hooks:
      - id: isort
        args: [--profile=black]

  - repo: https://github.com/pycqa/flake8
    rev: 6.0.0
    hooks:
      - id: flake8
        args: [--max-line-length=88, --ignore=E203,W503]

  - repo: local
    hooks:
      - id: ai-code-review
        name: AI Code Review
        entry: python scripts/ai_code_review.py
        language: python
        files: \.py$
        stages: [pre-commit]

# pyproject.toml - 统一工具配置
[tool.black]
line-length = 88
target-version = ['py39']
include = '\.pyi?$'

[tool.isort]
profile = "black"
multi_line_output = 3
line_length = 88

[tool.mypy]
python_version = "3.9"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true

[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = "test_*.py"
python_classes = "Test*"
addopts = "-v --tb=short --strict-markers"
```

### 3. AI协作工作流

```
Human: 我们团队需要实现用户权限管理功能。让我们按照团队协作流程来进行。

**第一步：架构设计讨论**

Team Lead: 我来设计权限管理的整体架构：

```python
# 权限管理模块架构设计
"""
permissions/
├── models/
│   ├── role.py      # 角色模型
│   ├── permission.py # 权限模型  
│   └── user_role.py # 用户角色关联
├── services/
│   ├── role_service.py       # 角色服务
│   ├── permission_service.py # 权限服务
│   └── access_control.py     # 访问控制
├── decorators/
│   └── auth_required.py # 权限装饰器
└── tests/
    └── test_permissions.py
"""

class PermissionArchitecture:
    """权限管理架构定义"""
    
    def __init__(self):
        self.roles = {}  # 角色定义
        self.permissions = {}  # 权限定义
        self.user_roles = {}  # 用户角色映射
    
    def define_roles(self):
        """定义系统角色"""
        return {
            'admin': ['user:create', 'user:read', 'user:update', 'user:delete'],
            'manager': ['user:read', 'user:update', 'order:manage'],
            'user': ['profile:read', 'profile:update', 'order:create']
        }
    
    def define_permissions(self):
        """定义权限粒度"""
        return {
            'user:create': '创建用户',
            'user:read': '查看用户',
            'user:update': '更新用户',
            'user:delete': '删除用户',
            'order:create': '创建订单',
            'order:manage': '管理订单'
        }
```

**第二步：任务分配**

Team Lead: 现在分配开发任务：
- @Alice: 实现角色模型和服务
- @Bob: 实现权限装饰器和访问控制
- @Charlie: 编写测试用例
- AI助手: 协助各成员实现具体代码
```

### 4. 代码审查协作

```python
# scripts/team_code_review.py
"""团队代码审查辅助工具"""

import subprocess
import json
from typing import List, Dict
from dataclasses import dataclass

@dataclass
class CodeReviewComment:
    file_path: str
    line_number: int
    severity: str  # 'info', 'warning', 'error'
    message: str
    reviewer: str

class TeamCodeReviewer:
    """团队代码审查器"""
    
    def __init__(self):
        self.team_standards = self._load_team_standards()
    
    def _load_team_standards(self) -> Dict:
        """加载团队编码标准"""
        return {
            'max_function_length': 50,
            'max_class_length': 200,
            'required_docstrings': True,
            'type_hints_required': True,
            'test_coverage_min': 80
        }
    
    def review_ai_generated_code(self, file_path: str) -> List[CodeReviewComment]:
        """审查AI生成的代码"""
        comments = []
        
        with open(file_path, 'r') as f:
            lines = f.readlines()
        
        # 检查函数长度
        comments.extend(self._check_function_length(file_path, lines))
        
        # 检查文档字符串
        comments.extend(self._check_docstrings(file_path, lines))
        
        # 检查类型注解
        comments.extend(self._check_type_hints(file_path, lines))
        
        return comments
    
    def _check_function_length(self, file_path: str, lines: List[str]) -> List[CodeReviewComment]:
        """检查函数长度"""
        comments = []
        current_function = None
        function_start = 0
        
        for i, line in enumerate(lines):
            if line.strip().startswith('def '):
                if current_function:
                    # 检查上一个函数的长度
                    length = i - function_start
                    if length > self.team_standards['max_function_length']:
                        comments.append(CodeReviewComment(
                            file_path=file_path,
                            line_number=function_start + 1,
                            severity='warning',
                            message=f'Function {current_function} is too long ({length} lines). Consider breaking it down.',
                            reviewer='AI_REVIEWER'
                        ))
                
                current_function = line.strip().split('(')[0].replace('def ', '')
                function_start = i
        
        return comments
    
    def _check_docstrings(self, file_path: str, lines: List[str]) -> List[CodeReviewComment]:
        """检查文档字符串"""
        comments = []
        
        for i, line in enumerate(lines):
            if line.strip().startswith('def ') or line.strip().startswith('class '):
                # 检查下一行是否有文档字符串
                if i + 1 < len(lines):
                    next_line = lines[i + 1].strip()
                    if not next_line.startswith('"""') and not next_line.startswith("'''"):
                        function_name = line.strip().split('(')[0].replace('def ', '').replace('class ', '')
                        comments.append(CodeReviewComment(
                            file_path=file_path,
                            line_number=i + 1,
                            severity='warning',
                            message=f'{function_name} is missing docstring',
                            reviewer='AI_REVIEWER'
                        ))
        
        return comments

# 集成到GitHub Actions中
def generate_review_report(pr_files: List[str]) -> Dict:
    """为PR生成审查报告"""
    reviewer = TeamCodeReviewer()
    all_comments = []
    
    for file_path in pr_files:
        if file_path.endswith('.py'):
            comments = reviewer.review_ai_generated_code(file_path)
            all_comments.extend(comments)
    
    # 生成报告
    report = {
        'total_files': len(pr_files),
        'python_files': len([f for f in pr_files if f.endswith('.py')]),
        'total_issues': len(all_comments),
        'by_severity': {
            'error': len([c for c in all_comments if c.severity == 'error']),
            'warning': len([c for c in all_comments if c.severity == 'warning']),
            'info': len([c for c in all_comments if c.severity == 'info'])
        },
        'comments': [
            {
                'file': c.file_path,
                'line': c.line_number,
                'severity': c.severity,
                'message': c.message
            }
            for c in all_comments
        ]
    }
    
    return report
```

### 5. 团队知识共享

```python
# knowledge_base/ai_coding_patterns.py
"""团队AI编程模式库"""

class TeamCodingPatterns:
    """团队编程模式集合"""
    
    @staticmethod
    def get_user_service_pattern():
        """用户服务标准模式"""
        return {
            'description': '用户服务实现标准',
            'template': '''
class UserService:
    """用户服务 - 团队标准实现"""
    
    def __init__(self, user_repository: UserRepositoryInterface):
        self.user_repository = user_repository
        self.logger = logging.getLogger(__name__)
    
    async def create_user(self, user_data: CreateUserRequest) -> UserResponse:
        """创建用户 - 标准流程"""
        try:
            # 1. 输入验证
            self._validate_user_data(user_data)
            
            # 2. 业务逻辑检查
            await self._check_business_rules(user_data)
            
            # 3. 执行创建
            user = await self.user_repository.create(user_data)
            
            # 4. 发布事件
            await self._publish_user_created_event(user)
            
            # 5. 返回结果
            return UserResponse.from_model(user)
            
        except Exception as e:
            self.logger.error(f"Failed to create user: {e}")
            raise
            ''',
            'usage_notes': [
                '总是使用类型注解',
                '包含完整的错误处理',
                '记录关键操作日志',
                '发布领域事件',
                '使用标准的响应格式'
            ]
        }
    
    @staticmethod
    def get_api_controller_pattern():
        """API控制器标准模式"""
        return {
            'description': 'REST API控制器标准',
            'template': '''
from fastapi import APIRouter, Depends, HTTPException, status
from app.schemas import CreateUserRequest, UserResponse
from app.services import UserService

router = APIRouter(prefix="/users", tags=["users"])

@router.post("/", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def create_user(
    user_data: CreateUserRequest,
    user_service: UserService = Depends()
) -> UserResponse:
    """
    创建新用户
    
    - **user_data**: 用户创建请求数据
    - **return**: 创建的用户信息
    """
    try:
        return await user_service.create_user(user_data)
    except ValidationError as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=str(e)
        )
    except Exception as e:
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail="Internal server error"
        )
            ''',
            'requirements': [
                '使用依赖注入',
                '标准HTTP状态码',
                '完整的API文档',
                '适当的异常处理',
                '类型安全的响应'
            ]
        }

# 团队提示词库
TEAM_PROMPTS = {
    'create_service': """
请基于团队标准创建{service_name}服务：

**必须遵循的模式：**
{pattern_template}

**团队要求：**
- 使用依赖注入
- 包含完整的类型注解
- 实现异步方法（如需要）
- 添加详细的文档字符串
- 包含错误处理和日志记录

**不要修改：**
- 现有的接口定义
- 其他服务的实现
- 数据库模型结构
""",
    
    'fix_bug_safely': """
请安全地修复以下bug：

**Bug信息：**{bug_info}

**安全约束：**
- 只修改最小必要的代码
- 不要重构现有功能
- 保持API接口兼容性
- 添加回归测试

**团队标准：**
- 修复后运行完整测试套件
- 更新相关文档
- 记录修复日志
"""
}
```

### 6. 冲突解决机制

```python
# scripts/conflict_resolver.py
"""AI代码冲突解决工具"""

class AICodeConflictResolver:
    """AI代码冲突解决器"""
    
    def analyze_merge_conflict(self, conflict_file: str) -> Dict:
        """分析合并冲突"""
        with open(conflict_file, 'r') as f:
            content = f.read()
        
        conflicts = self._parse_conflicts(content)
        analysis = {
            'total_conflicts': len(conflicts),
            'conflict_types': self._categorize_conflicts(conflicts),
            'resolution_suggestions': self._suggest_resolutions(conflicts)
        }
        
        return analysis
    
    def _parse_conflicts(self, content: str) -> List[Dict]:
        """解析Git冲突标记"""
        conflicts = []
        lines = content.split('\n')
        
        i = 0
        while i < len(lines):
            if lines[i].startswith('<<<<<<<'):
                conflict = self._extract_conflict_block(lines, i)
                conflicts.append(conflict)
                i = conflict['end_line']
            i += 1
        
        return conflicts
    
    def _suggest_resolutions(self, conflicts: List[Dict]) -> List[Dict]:
        """建议冲突解决方案"""
        suggestions = []
        
        for conflict in conflicts:
            if self._is_formatting_conflict(conflict):
                suggestions.append({
                    'type': 'formatting',
                    'action': 'auto_format',
                    'description': '使用团队代码格式化工具自动解决'
                })
            elif self._is_import_conflict(conflict):
                suggestions.append({
                    'type': 'imports',
                    'action': 'merge_and_sort',
                    'description': '合并导入语句并排序'
                })
            else:
                suggestions.append({
                    'type': 'manual',
                    'action': 'review_required',
                    'description': '需要人工审查和决策'
                })
        
        return suggestions

# 团队冲突解决工作流
def resolve_ai_conflicts():
    """团队AI冲突解决流程"""
    
    # 1. 检测冲突
    conflicts = subprocess.run(['git', 'diff', '--name-only', '--diff-filter=U'], 
                              capture_output=True, text=True)
    
    if not conflicts.stdout.strip():
        print("✅ 没有发现冲突")
        return
    
    conflict_files = conflicts.stdout.strip().split('\n')
    
    # 2. 分析每个冲突文件
    resolver = AICodeConflictResolver()
    
    for file_path in conflict_files:
        print(f"\n🔍 分析冲突文件: {file_path}")
        
        analysis = resolver.analyze_merge_conflict(file_path)
        
        print(f"   冲突数量: {analysis['total_conflicts']}")
        print(f"   冲突类型: {analysis['conflict_types']}")
        
        # 3. 应用自动解决方案
        for suggestion in analysis['resolution_suggestions']:
            if suggestion['action'] == 'auto_format':
                print(f"   🔧 自动格式化解决")
                subprocess.run(['black', file_path])
                subprocess.run(['isort', file_path])
            
            elif suggestion['action'] == 'merge_and_sort':
                print(f"   📦 合并导入语句")
                # 实现导入语句合并逻辑
                
            else:
                print(f"   ⚠️  需要人工解决: {suggestion['description']}")

if __name__ == "__main__":
    resolve_ai_conflicts()
```

## 团队协作总结

### ✅ 团队AI Coding检查清单

**工具标准化：**
- [ ] 统一AI工具选择
- [ ] 配置一致的代码格式化
- [ ] 建立标准提示词库
- [ ] 设置自动化质量检查

**协作流程：**
- [ ] 定义清晰的任务分配
- [ ] 建立代码审查标准
- [ ] 实现冲突解决机制
- [ ] 维护团队知识库

**质量保证：**
- [ ] 自动化测试和CI/CD
- [ ] 定期代码质量评估
- [ ] 团队培训和知识分享
- [ ] 持续改进协作流程

通过规范化的团队协作，可以最大化AI Coding的团队效益，减少协作摩擦。