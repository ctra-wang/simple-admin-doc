---
order: 12
title: '权限验证'
---

# 权限验证

我们使用Casbin进行权限验证.

## 初始化

### 使用 redis 监控权限变化

```go
cbn := c.CasbinConf.MustNewCasbinWithRedisWatcher(c.DatabaseConf.Type, c.DatabaseConf.GetDSN(), c.RedisConf)
```

## 创建中间件

```go
package middleware

import (
	"context"
	"net/http"
	"strings"

	"github.com/casbin/casbin/v2"
	"github.com/suyuan32/simple-admin-common/enum/errorcode"
	"github.com/zeromicro/go-zero/core/errorx"
	"github.com/zeromicro/go-zero/core/logx"
	"github.com/zeromicro/go-zero/core/stores/redis"
	"github.com/zeromicro/go-zero/rest/httpx"

	"github.com/suyuan32/simple-admin-common/i18n"
)

type AuthorityMiddleware struct {
	Cbn   *casbin.Enforcer
	Rds   *redis.Redis
	Trans *i18n.Translator
}

func NewAuthorityMiddleware(cbn *casbin.Enforcer, rds *redis.Redis, trans *i18n.Translator) *AuthorityMiddleware {
	return &AuthorityMiddleware{
		Cbn:   cbn,
		Rds:   rds,
		Trans: trans,
	}
}

func (m *AuthorityMiddleware) Handle(next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		// get the path
		obj := r.URL.Path
		// get the method
		act := r.Method
		// get the role id
		roleIds := r.Context().Value("roleId").(string)

		// check jwt blacklist
		jwtResult, err := m.Rds.Get("token_" + r.Header.Get("Authorization"))
		if err != nil {
			logx.Errorw("redis error in jwt", logx.Field("detail", err.Error()))
			httpx.Error(w, errorx.NewApiError(http.StatusInternalServerError, err.Error()))
			return
		}
		if jwtResult == "1" {
			logx.Errorw("token in blacklist", logx.Field("detail", r.Header.Get("Authorization")))
			httpx.Error(w, errorx.NewApiErrorWithoutMsg(http.StatusUnauthorized))
			return
		}

		result := batchCheck(m.Cbn, roleIds, act, obj)

		if result {
			logx.Infow("HTTP/HTTPS Request", logx.Field("UUID", r.Context().Value("userId").(string)),
				logx.Field("path", obj), logx.Field("method", act))
			next(w, r)
			return
		} else {
			logx.Errorw("the role is not permitted to access the API", logx.Field("roleId", roleIds),
				logx.Field("path", obj), logx.Field("method", act))
			httpx.Error(w, errorx.NewCodeError(errorcode.PermissionDenied, m.Trans.Trans(
				context.WithValue(context.Background(), "lang", r.Header.Get("Accept-Language")),
				"common.permissionDeny")))
			return
		}
	}
}

func batchCheck(cbn *casbin.Enforcer, roleIds, act, obj string) bool {
	var checkReq [][]any
	for _, v := range strings.Split(roleIds, ",") {
		checkReq = append(checkReq, []any{v, obj, act})
	}

	result, err := cbn.BatchEnforce(checkReq)
	if err != nil {
		logx.Errorw("Casbin enforce error", logx.Field("detail", err.Error()))
		return false
	}

	for _, v := range result {
		if v {
			return true
		}
	}

	return false
}

```

[Middleware](https://github.com/suyuan32/simple-admin-core/blob/master/api/internal/middleware/authoritymiddleware.go)

## 在 api 文件中使用

```text
@server(
    jwt: Auth
    group: api
    middleware: Authority
)
```

[如何使用中间件](https://go-zero.dev/cn/docs/advance/middleware)

::: info
目前API生成已支持自动生成鉴权中间件
:::