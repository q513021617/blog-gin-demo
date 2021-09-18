# 创建项目

安装Gin 框架
```
go get github.com/gin-gonic/gin
```
安装gin
```
go get github.com/codegangsta/gin
```


项目运行

```
gin -p 8080 -a 9090 —all run
```

# 配置文件修改

```
[app]
PageSize = 10
JwtSecret = 233
PrefixUrl = http://127.0.0.1:8000

RuntimeRootPath = runtime/

ImageSavePath = upload/images/
# MB
ImageMaxSize = 5
ImageAllowExts = .jpg,.jpeg,.png

ExportSavePath = export/
QrCodeSavePath = qrcode/
FontSavePath = fonts/

LogSavePath = logs/
LogSaveName = log
LogFileExt = log
TimeFormat = 20060102

[server]
#debug or release
RunMode = debug
HttpPort = 8000
ReadTimeout = 60
WriteTimeout = 60

[database]
Type = mysql
User = root
Password = root
Host = 127.0.0.1:3306
Name = blog
TablePrefix = blog_

[redis]
Host = 127.0.0.1:6379
Password =
MaxIdle = 30
MaxActive = 30
IdleTimeout = 200
```

# 创建程序入口

main.go
```
package main

import (
	"fmt"
	"log"
	"net/http"

	"github.com/gin-gonic/gin"

	"github.com/EDDYCJY/go-gin-example/models"
	"github.com/EDDYCJY/go-gin-example/pkg/gredis"
	"github.com/EDDYCJY/go-gin-example/pkg/logging"
	"github.com/EDDYCJY/go-gin-example/pkg/setting"
	"github.com/EDDYCJY/go-gin-example/routers"
	"github.com/EDDYCJY/go-gin-example/pkg/util"
)

func init() {
	setting.Setup()
	models.Setup()
	logging.Setup()
	gredis.Setup()
	util.Setup()
}

// @license.url https://github.com/EDDYCJY/go-gin-example/blob/master/LICENSE
func main() {
	gin.SetMode(setting.ServerSetting.RunMode)

	routersInit := routers.InitRouter()
	readTimeout := setting.ServerSetting.ReadTimeout
	writeTimeout := setting.ServerSetting.WriteTimeout
	endPoint := fmt.Sprintf(":%d", setting.ServerSetting.HttpPort)
	maxHeaderBytes := 1 << 20

	server := &http.Server{
		Addr:           endPoint,
		Handler:        routersInit,
		ReadTimeout:    readTimeout,
		WriteTimeout:   writeTimeout,
		MaxHeaderBytes: maxHeaderBytes,
	}

	log.Printf("[info] start http server listening %s", endPoint)

	server.ListenAndServe()

}

```

# 创建路由

```
package routers

import (
	"net/http"
	"github.com/gin-gonic/gin"
	_ "github.com/EDDYCJY/go-gin-example/docs"
	"github.com/swaggo/gin-swagger"
	"github.com/swaggo/gin-swagger/swaggerFiles"
	"github.com/EDDYCJY/go-gin-example/middleware/jwt"
	"github.com/EDDYCJY/go-gin-example/routers/api"
	"github.com/EDDYCJY/go-gin-example/routers/api/v1"
)

// 初始化路由信息
func InitRouter() *gin.Engine {
	r := gin.New() //创建gin对象
	r.Use(gin.Logger()) //设置日志器
	r.Use(gin.Recovery()) //设置异常熔断，遇到异常时不会停止程序
	r.StaticFS("/export", http.Dir(export.GetExcelFullPath())) //映射静态资源文件的请求路径
	r.POST("/auth", api.GetAuth) //映射请求路径和处理方法
	r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler)) //映射swaggger的请求路径

	apiv1 := r.Group("/api/v1") //设置api分组
	apiv1.Use(jwt.JWT())  //该分组使用JWT验证
	{
		//获取标签列表
		apiv1.GET("/tags", v1.GetTags) // 映射请求路径和处理方法
		//新建标签
		apiv1.POST("/tags", v1.AddTag)
		//更新指定标签
		apiv1.PUT("/tags/:id", v1.EditTag)
		//删除指定标签
		apiv1.DELETE("/tags/:id", v1.DeleteTag)
	}

	return r
}
```
# 创建请求方法
```
func GetTags(c *gin.Context) {
	appG := app.Gin{C: c} //接收gin对象指针并赋值给新变量
	name := c.Query("name") //获取参数name的值并赋值给变量name
	state := -1 //状态设置为-1
	if arg := c.Query("state"); arg != "" {  //获取state参数并赋值，并判断不为空
		state = com.StrTo(arg).MustInt() // 转换字符串为整数
	}

	tagService := tag_service.Tag{
		Name:     name,
		State:    state,
		PageNum:  util.GetPage(c),
		PageSize: setting.AppSetting.PageSize,
	}//将查询条件赋值给 tagService 
	tags, err := tagService.GetAll() // 按照之前的查询条件查询所有数据
	if err != nil { //如果有异常则服务器向浏览器抛出异常
		appG.Response(http.StatusInternalServerError, e.ERROR_GET_TAGS_FAIL, nil)
		return
	}

	count, err := tagService.Count() //获取记录条数
	if err != nil {
		appG.Response(http.StatusInternalServerError, e.ERROR_COUNT_TAG_FAIL, nil)
		return
	}
// 服务器响应给浏览器数据
	appG.Response(http.StatusOK, e.SUCCESS, map[string]interface{}{
		"lists": tags,
		"total": count,
	})
}

```

# 创建service层
```
package tag_service

import (
	"encoding/json"
	"io"
	"strconv"
	"time"

	"github.com/360EntSecGroup-Skylar/excelize"
	"github.com/tealeg/xlsx"

	"github.com/EDDYCJY/go-gin-example/models"
	"github.com/EDDYCJY/go-gin-example/pkg/export"
	"github.com/EDDYCJY/go-gin-example/pkg/file"
	"github.com/EDDYCJY/go-gin-example/pkg/gredis"
	"github.com/EDDYCJY/go-gin-example/pkg/logging"
	"github.com/EDDYCJY/go-gin-example/service/cache_service"
)
//这里的 struct 只是作为查询条件使用的，可以理解为Java中的dto
type Tag struct {
	ID         int
	Name       string
	CreatedBy  string
	ModifiedBy string
	State      int

	PageNum  int
	PageSize int
}

//具体的 查询逻辑，但是数据层逻辑不在这里

func (t *Tag) GetAll() ([]models.Tag, error) {
	var (
		tags, cacheTags []models.Tag
	)
// 赋值条件
	cache := cache_service.Tag{
		State: t.State,

		PageNum:  t.PageNum,
		PageSize: t.PageSize,
	}
// 将条件拼接为一个字符串
	key := cache.GetTagsKey()
// 先查看redis是否存在缓存
if gredis.Exists(key) {
// 如果存在则从redis里面获取
		data, err := gredis.Get(key)
		if err != nil {
			logging.Info(err)
		} else {
//将json字符串序列化为对象并赋值给目标对象
			json.Unmarshal(data, &cacheTags)
			return cacheTags, nil
		}
	}
// 条件查询获取数据库中符合条件的tags,条件yongmap表示
	tags, err := models.GetTags(t.PageNum, t.PageSize, t.getMaps())
	if err != nil {
		return nil, err
	}
// 设置新的缓存
	gredis.Set(key, tags, 3600)
	return tags, nil
}

```

# 创建cacheService 
```
package cache_service

import (
	"strconv"
	"strings"
	"github.com/EDDYCJY/go-gin-example/pkg/e"
)

type Tag struct {
	ID    int
	Name  string
	State int
	PageNum  int
	PageSize int
}
// 这大概就是通过一个string数组来作为服务器缓存
func (t *Tag) GetTagsKey() string {
// 声明一个字符串数组,并先放入两个元素 e.CACHE_TAG,"LIST"
keys := []string{
		e.CACHE_TAG,
		"LIST",
	}

	if t.Name != "" {
		keys = append(keys, t.Name)
	}
	if t.State >= 0 {
		keys = append(keys, strconv.Itoa(t.State))
	}
	if t.PageNum > 0 {
		keys = append(keys, strconv.Itoa(t.PageNum))
	}
	if t.PageSize > 0 {
		keys = append(keys, strconv.Itoa(t.PageSize))
	}

	return strings.Join(keys, "_")
}

```

# 创建model层


```
package models

import (
	"github.com/jinzhu/gorm"
)

```

```
type Tag struct {
	Model

	Name       string `json:"name"`
	CreatedBy  string `json:"created_by"`
	ModifiedBy string `json:"modified_by"`
	State      int    `json:"state"`
}

```



```
func GetTags(pageNum int, pageSize int, maps interface{}) ([]Tag, error) {
	var (
		tags []Tag
		err  error
	)

	if pageSize > 0 && pageNum > 0 {
		err = db.Where(maps).Find(&tags).Offset(pageNum).Limit(pageSize).Error
	} else {
		err = db.Where(maps).Find(&tags).Error
	}

	if err != nil && err != gorm.ErrRecordNotFound {
		return nil, err
	}

	return tags, nil
}

```


# 总结

至此完成了gin框架的CRUD，但是权限认证等功能均没有介绍,详细请看
项目地址: https://github.com/q513021617/blog-gin-demo