# Go 快速入门（对比 Node.js）

> 写给从 Node.js 转 Go 的工程师。假设你已经熟悉 JavaScript/TypeScript 基础。

---

## 1. 核心差异

| 方面 | Node.js | Go |
|------|---------|-----|
| 类型系统 | 弱类型（可用 any） | **强类型**，每个变量有明确类型 |
| 并发模型 | 单线程事件循环 | **Goroutine**（轻量级线程） |
| 错误处理 | try/catch | **返回值错误**（多返回值函数） |
| 包管理 | npm | go mod |
| 主流框架 | Express | **Gin** (类似 Express) |
| ORM | Sequelize/Prisma | **GORM** |

---

## 2. 基础语法对比

### 2.1 变量声明

```go
// Node.js
let name = "John";
const age = 25;

// Go
var name string = "John"      // 标准声明
age := 25                      // 简短声明（函数内）
const PI = 3.14                // 常量
```

### 2.2 函数

```go
// Node.js
function add(a, b) {
  return a + b;
}

// Go
func add(a int, b int) int {
  return a + b
}

// 多返回值（Go 的错误处理方式）
func divide(a, b int) (int, error) {
  if b == 0 {
    return 0, errors.New("division by zero")
  }
  return a / b, nil
}
```

### 2.3 错误处理

```go
// Node.js
try {
  const result = await someAsyncFunc();
} catch (err) {
  console.error(err);
}

// Go
result, err := someAsyncFunc()
if err != nil {
  log.Fatal(err)  // 或返回错误给调用者
}
```

### 2.4 Struct 和 JSON

```go
// 定义结构体
type User struct {
  ID   uint   `json:"id"`
  Name string `json:"name"`
  Age  int    `json:"age"`
}

// Node.js (TypeScript)
interface User {
  id: number;
  name: string;
  age: number;
}
```

### 2.5 GORM 使用（类比 Sequelize）

```go
// 定义模型
type Product struct {
  gorm.Model              // 内含 ID, CreatedAt, UpdatedAt, DeletedAt
  Name  string
  Price float64
}

// 创建记录
db.Create(&product)

// 查询
var product Product
db.First(&product, 1)           // 按 ID 查
db.Where("name = ?", "demo").First(&product)

// 更新
db.Model(&product).Update("price", 100)

// 删除
db.Delete(&product)

// Node.js (Sequelize)
Product.create({ name: "demo", price: 100 })
Product.findByPk(1)
Product.update({ price: 100 }, { where: { id: 1 } })
Product.destroy({ where: { id: 1 } })
```

---

## 3. Gin 路由（类比 Express）

```go
import "github.com/gin-gonic/gin"

// 创建路由
r := gin.Default()

// GET 请求
r.GET("/users", func(c *gin.Context) {
  c.JSON(200, gin.H{"users": []string{"Tom", "Jerry"}})
})

// POST 请求
r.POST("/users", func(c *gin.Context) {
  var user User
  c.ShouldBindJSON(&user)  // 解析 JSON body
  c.JSON(201, user)
})

// 路由组
v1 := r.Group("/api/v1")
v1.GET("/products", getProducts)
v1.POST("/products", createProduct)

// 启动服务
r.Run(":8080")

// Node.js/Express 对比
// app.get('/users', (req, res) => res.json({users: ['Tom', 'Jerry']}))
// app.post('/users', (req, res) => { const user = req.body; res.status(201).json(user); })
```

---

## 4. JWT 认证（Gin 中间件）

```go
// Node.js Express
// const authMiddleware = require('./middleware/jwt')
// app.use('/api', authMiddleware)

// Go Gin
import "github.com/gin-gonic/gin"
import "github.com/golang-jwt/jwt/v5"

func AuthMiddleware() gin.HandlerFunc {
  return func(c *gin.Context) {
    token := c.GetHeader("Authorization")
    if token == "" {
      c.JSON(401, gin.H{"error": "unauthorized"})
      c.Abort()
      return
    }
    // 验证 JWT...
    c.Set("userID", userID)
    c.Next()
  }
}

r.Use(AuthMiddleware())
```

---

## 5. 项目启动命令

```bash
# Node.js
npm install
npm run dev

# Go
go mod download
go run main.go
```

---

## 6. 常用命令速查

| 操作 | Node.js | Go |
|------|---------|-----|
| 依赖安装 | `npm install` | `go mod tidy` |
| 启动开发 | `npm run dev` | `go run ./cmd/api` |
| 构建 | `npm run build` | `go build -o api ./cmd/api` |
| 测试 | `npm test` | `go test ./...` |
| 格式化 | `prettier` | `go fmt` |
