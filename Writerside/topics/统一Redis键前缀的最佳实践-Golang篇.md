#  统一Redis键前缀的最佳实践(Golang篇)

日常开发中遇到了一个特殊的需求，给所有的redis key添加统一的前缀。

因为项目里面使用的客户端是 `go-redis`，于是调研了下，发现它允许配置 Hook

参考链接 **[Go Redis Hook 钩子](https://redis.uptrace.dev/zh/guide/go-redis-hook.html)**

于是自己动手实现了下， 关键在于实现`ProcessHook`和`ProcessPipelineHook`这两个方法

```go
func (h AppPrefixHook) ProcessHook(next redis.ProcessHook) redis.ProcessHook {
	return func(ctx context.Context, cmd redis.Cmder) error {
		// 有些key不需要添加前缀，这里加个判断
		if !shouldSkipPrefix(ctx) {
		    // 实际添加前缀的功能被封装在了addPrefixToArgs方法里
			h.addPrefixToArgs(ctx, cmd)
		}
		return next(ctx, cmd)
	}
}

// ProcessPipelineHook 实现方式跟上面类似，这里省略
```
下面是addPrefixToArgs方法的实现，篇幅问题，简化一下
```Go
var commandsWithPrefix = []string{"GET", "SET"}

func (h AppPrefixHook) addPrefixToArgs(ctx context.Context, cmd redis.Cmder) {
	// directly change the args variable, because the memory address is the same
	args := cmd.Args()
	if len(args) <= 1 {
		return
	}

	name := strings.ToUpper(cmd.Name())
	switch name {
	case "DEL", "EXISTS":
		// common multi `key` command: DEL key1 key2 key3
		for i := 1; i < len(args); i++ {
			args[i] = h.Prefix + cast.ToString(args[i])
		}
	default:
		if lo.IndexOf[string](commandsWithPrefix, name) != -1 {
			args[1] = h.Prefix + cast.ToString(args[1])
		} else {
			fmt.Println("unsupport app prefix command: ", name)
		}
	}
}
```
实际上，可能有些key并不需要添加统一前缀，所以我采用了一种更灵活的实现方式：
通过添加context钩子，允许调用者根据实际需求，动态控制哪些key跳过该逻辑。
```Go
type contextKey string

const skipPrefixKey contextKey = "skip"

// WithSkipPrefix define a context helper function, when a key does not need a prefix, use this function, example: Cli.Set(WithSkipPrefix(ctx), "key", "value")
func WithSkipPrefix(ctx context.Context) context.Context {
	return context.WithValue(ctx, skipPrefixKey, true)
}

func shouldSkipPrefix(ctx context.Context) bool {
	value := ctx.Value(skipPrefixKey)
	skip, ok := value.(bool)
	return ok && skip
}
```
最后， 对于调用者来说，只需要在初始化redis客户端的时候，把hook加上就行，全部下来也只需改动一行代码
```Go
package main

import (
	"context"
	"github.com/redis/go-redis/v9"
	"github.com/teaGod-s/go-redis-prefix"
)

func main() {
	Cli := redis.NewUniversalClient(&redis.UniversalOptions{
		Addrs:    []string{"localhost:6379"},
		Password: "",
	})

    // 添加钩子，并指定你喜欢的前缀，后面就可以愉快的使用啦！
	Cli.AddHook(prefix.AppPrefixHook{Prefix: "prefix4k:"})

	ctx := context.Background()
	
	// a example for the key with prefix
	// GET prefix4k:hello
	Cli.Get(ctx, "hello")

	// a example for the key with no prefix
	// GET hello
	Cli.Get(prefix.WithSkipPrefix(ctx), "hello")
}
```
完整代码已经上传到我的 **[Github仓库](https://github.com/teaGod-s/go-redis-prefix)**，如果对你有用的话，可以点一个star