# URL(路由)分发机制

## URLconf

Django把urlpatterns变量中的列表视为URLconf，urlpatterns本质是URL以及为该URL调用的视图函数之间的映射表。  
```python
urlpatterns = [
    path('admin/', admin.site.urls),
    path('index/', views.index), # 普通路径
    re_path(r'^articles/([0-9]{4})/$', views.articles), # 正则路径
]
```
导入其他的URL文件：  
```python
urlpatterns = [
        path('login/', include('app.urls'))#假设自己新建的urls在app（应用中）
    ]
```
命名URL：  
```python
urlpatterns = [
   path('articles/<int:year>/', views.year_archive, name='news-year-archive'),
]
```
URL的反向解析：  
在Python代码中可以通过reverse函数查找任何url的名称并返回其对应路径  
```python
return HttpResponseRedirect(reverse('news-year-archive', args=(year,))
```
URLconf不检查请求的方法，同一个URL的【POST】、【GET】、【HEAD】等请求都将路由到相同的函数。  

## 处理请求

Django需要确定使用的根URLconf，通常是ROOT_URLCONF设置的值，如果传入HttpRequest对象具有一个urlconf属性（由中间件设置），则其值将用于代替ROOT_URLCONF设置。
```python
# django/core/handlers/base.py
def resolve_request(self, request):
    """
    Retrieve/set the urlconf for the request. Return the view resolved,
    with its args and kwargs.
    """
    # Work out the resolver.
    if hasattr(request, 'urlconf'):
        urlconf = request.urlconf
        set_urlconf(urlconf)
        resolver = get_resolver(urlconf)
    else:
        resolver = get_resolver()
    # Resolve the view, and assign the match object back to the request.
    resolver_match = resolver.resolve(request.path_info)
    request.resolver_match = resolver_match
    return resolver_match

def get_resolver(urlconf=None):
    if urlconf is None:
        urlconf = settings.ROOT_URLCONF
    return _get_cached_resolver(urlconf)


@functools.lru_cache(maxsize=None)
def _get_cached_resolver(urlconf=None):
    return URLResolver(RegexPattern(r'^/'), urlconf)
```

通过URLResolver的resolve方法解析url，匹配成功则返回ResolverMatch。
```python
def resolve(self, path):
    path = str(path)  # path may be a reverse_lazy object
    tried = []
    match = self.pattern.match(path)
    if match:
        new_path, args, kwargs = match
        for pattern in self.url_patterns:
            try:
                sub_match = pattern.resolve(new_path)
            except Resolver404 as e:
                sub_tried = e.args[0].get('tried')
                if sub_tried is not None:
                    tried.extend([pattern] + t for t in sub_tried)
                else:
                    tried.append([pattern])
            else:
                if sub_match:
                    # Merge captured arguments in match with submatch
                    sub_match_dict = {**kwargs, **self.default_kwargs}
                    # Update the sub_match_dict with the kwargs from the sub_match.
                    sub_match_dict.update(sub_match.kwargs)
                    # If there are *any* named groups, ignore all non-named groups.
                    # Otherwise, pass all non-named arguments as positional arguments.
                    sub_match_args = sub_match.args
                    if not sub_match_dict:
                        sub_match_args = args + sub_match.args
                    current_route = '' if isinstance(pattern, URLPattern) else str(pattern.pattern)
                    return ResolverMatch(
                        sub_match.func,
                        sub_match_args,
                        sub_match_dict,
                        sub_match.url_name,
                        [self.app_name] + sub_match.app_names,
                        [self.namespace] + sub_match.namespaces,
                        self._join_route(current_route, sub_match.route),
                    )
                tried.append([pattern])
        raise Resolver404({'tried': tried, 'path': new_path})
    raise Resolver404({'path': path})
```

匹配成功后取出回调函数和参数，应用视图中间件，如果response为空，则执行wrapped_callback（即视图中的代码），返回response。
```python
def _get_response(self, request):
    """
    Resolve and call the view, then apply view, exception, and
    template_response middleware. This method is everything that happens
    inside the request/response middleware.
    """
    response = None
    callback, callback_args, callback_kwargs = self.resolve_request(request)

    # Apply view middleware
    for middleware_method in self._view_middleware:
        response = middleware_method(request, callback, callback_args, callback_kwargs)
        if response:
            break

    if response is None:
        wrapped_callback = self.make_view_atomic(callback)
        # If it is an asynchronous view, run it in a subthread.
        if asyncio.iscoroutinefunction(wrapped_callback):
            wrapped_callback = async_to_sync(wrapped_callback)
        try:
            response = wrapped_callback(request, *callback_args, **callback_kwargs)
        except Exception as e:
            response = self.process_exception_by_middleware(e, request)
            if response is None:
                raise

    # Complain if the view returned None (a common error).
    self.check_response(response, callback)

    # If the response supports deferred rendering, apply template
    # response middleware and then render the response
    if hasattr(response, 'render') and callable(response.render):
        for middleware_method in self._template_response_middleware:
            response = middleware_method(request, response)
            # Complain if the template response middleware returned None (a common error).
            self.check_response(
                response,
                middleware_method,
                name='%s.process_template_response' % (
                    middleware_method.__self__.__class__.__name__,
                )
            )
        try:
            response = response.render()
        except Exception as e:
            response = self.process_exception_by_middleware(e, request)
            if response is None:
                raise

    return response
```