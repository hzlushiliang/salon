# nginx指令-高级篇 #

__关键词: nginx 指令 变量 编译 http_log__

在上一期[nginx指令-初级篇](https://github.com/hzlushiliang/salon/blob/master/nginx/nginx指令-初级篇.md)的末尾，我们留了一个彩蛋，前段时间太忙了，现在将彩蛋在这一期展开

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

上面的片段是从nginx样本配置中截取的，想必大家都很熟悉，我们通过`log_format`指令来定义了自己的日志格式，其中每一条日志开头是客户端的ip地址，紧接着一个横杠，横杠后面是客户端用户名，本地时间戳...

按照`log_format`指令的设计，我们将该日志格式取名为`main`，由于格式比较长，可以将格式用一对对的单引号包裹起来，作为多个指令参数传递给`log_format`指令，由指令本身的转储接口对这些参数做进一步的分解，语义分析和转储，后续可以通过`main`来直接引用该日志格式，比如说下面的引用方式

    access_log /var/log/nginx/access.log main;

我们将访问日志的格式配置为`main`，`main`是日志格式的名称，就是对先前定义的日志格式的引用，回到日志格式本身的定义，`http_log`模块是如何知道`$remote_addr`指代的是客户端ip地址呢，又是如何知道`$status`指代的是http响应的状态码呢，这些其实是`http_log`模块事先设计好的，并不是`http_log`模块来适配我们，而是我们去适配`http_log`模块，我们在应用的时候，必须按照它的设计规范来使用

在`log_format`指令的转储接口中，会对全局的日志格式做一次查重

    for (i = 0; i < lmcf->formats.nelts; i++) {
        if (fmt[i].name.len == value[1].len
            && ngx_strcmp(fmt[i].name.data, value[1].data) == 0)
        {
            ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                               "duplicate \"log_format\" name \"%V\"",
                               &value[1]);
            return NGX_CONF_ERROR;
        }
    }

当发现格式名称有重复，那么直接返回`NGX_CONF_ERROR`，nginx主进程就直接退出了，不会正常启动

校验通过之后，开始对日志格式做变量编译，`log_format`指令的变量编译在nginx内核变量编译的基础上包了一层

以`$`字符开头的字符串看做一个变量，并且对改变量的每个字符做了限制，只能是大写字符，小写字符，数字或者下划线

    if (value[s].data[i] == '$') {
        if ((ch >= 'A' && ch <= 'Z')
            || (ch >= 'a' && ch <= 'z')
            || (ch >= '0' && ch <= '9')
            || ch == '_')
        {
            continue;
        }
    }

比如说`$remote_addr`就是一个变量，变量以`$`字符开头作为标志，后面的字符串作为变量名

以`$`字符开头，后面用一对大括号`{}`包裹起来的也可以看做合法变量

    if (value[s].data[i] == '{') {
        bracket = 1;
    }

    if (ch == '}' && bracket) {
        i++;
        bracket = 0;
        break;
    }

比如说`${remote_addr}`就是一个变量，它和`$remote_addr`所表达的意思完全一样

除去上面两种变量，其他字符串都按照常量字符串来处理，打印日志的时候复制一遍即可

    while (i < value[s].len && value[s].data[i] != '$') {
        i++;
    }

对于提取到的变量，`log_format`指令做了分类，一种是指令自带的变量，自带的变量有哪些呢

    static ngx_http_log_var_t  ngx_http_log_vars[] = {
    { ngx_string("pipe"), 1, ngx_http_log_pipe },
    { ngx_string("time_local"), sizeof("28/Sep/1970:12:00:00 +0600") - 1,
                          ngx_http_log_time },
    { ngx_string("time_iso8601"), sizeof("1970-09-28T12:00:00+06:00") - 1,
                          ngx_http_log_iso8601 },
    { ngx_string("msec"), NGX_TIME_T_LEN + 4, ngx_http_log_msec },
    { ngx_string("request_time"), NGX_TIME_T_LEN + 4,
                          ngx_http_log_request_time },
    { ngx_string("status"), NGX_INT_T_LEN, ngx_http_log_status },
    { ngx_string("bytes_sent"), NGX_OFF_T_LEN, ngx_http_log_bytes_sent },
    { ngx_string("body_bytes_sent"), NGX_OFF_T_LEN,
                          ngx_http_log_body_bytes_sent },
    { ngx_string("request_length"), NGX_SIZE_T_LEN,
                          ngx_http_log_request_length },

    { ngx_null_string, 0, NULL }
    };

上面这些自带变量的解析和处理由`log_format`指令自己提供接口

非自带变量，则走nginx内核的内置变量处理流程，比如说`$remote_addr`和`$http_x_forwarded_for`就不在自带变量列表中

    if (ngx_http_log_variable_compile(cf, op, &var) != NGX_OK) {
        return NGX_CONF_ERROR;
    }

对于非自带变量，`log_format`指令直接提取nginx内置变量的数组下标，后续打日志时，直接通过数组下标，从http请求中提取动态取值

    index = ngx_http_get_variable_index(cf, value);
    value = ngx_http_get_indexed_variable(r, op->data);

上面的第一个函数用于提取nginx内置变量的数组下标，在实现上，先提取核心模块的六边形结构体，如果发现当前变量尚未设置，则分配内存存储在六边形结构体中

    v = ngx_array_push(&cmcf->variables);
    v->index = cmcf->variables.nelts - 1;
    return v->index;

上面的第二个函数用于提取当前请求当前变量的动态取值，对于不同的http请求，客户端ip地址可能是不同的，所以`remote_addr`变量的取值也会不同，`remote_addr`变量名称存储在核心模块的六边形结构体中，这个是全局唯一的，但是不同的http请求，变量取值属于动态值，存储在请求结构体中，通过数组下标来索引

    if (v[index].get_handler(r, &r->variables[index], v[index].data) == NGX_OK)
    return &r->variables[index];

nginx内置变量名称到变量动态取值的映射是谁完成的呢，由上面的`get_handler`回调接口完成，这个回调接口由定义内置变量的http模块来实现，比如说`remote_addr`内置变量是有http核心模块导出的，它负责基于`remote_addr`变量名，从http请求中解析客户端ip地址，然后存储到请求结构体中，方便其他模块引用

所以任何指令如果要支持nginx内置变量，需要像`log_format`指令一样，在转储接口中按照自己的设计，提取变量名，接着使用nginx核心模块的内置变量机制，主动编译和使用该变量，比如说`$remote_addr`和`$http_x_forwarded_for`这些内置变量已经是默认支持的，我们可以在自己的http模块中直接使用它们，本人以前做过一个项目，就用到`$remote_addr`内置变量，提取客户端ip地址之后，再基于客户端ip地址来做流量调度

前面提到的内置变量相对比较简单，像客户端ip地址，客户端端口号，xff头部这些信息，都属于http协议的标准范畴，nginx核心模块已经帮我们做掉了，但是http请求中有一些非标准信息，比如说`query string`部分的变量取值，这种变量的提取稍有不同

比如说我们有一个http请求

    http://127.0.0.1/car?name=bmw

为了在nginx模块中提取name的取值，我们需要在指令处理阶段编译该变量`$arg_name`，生成控制信息存储在自己的结构体中

    ngx_http_script_compile_t  sc;
    if (ngx_http_script_compile(&sc) != NGX_OK)


最后在http请求处理阶段，调nginx内核接口来提取动态值

    if (ngx_http_script_run(r, &name, llcf->name_lengths->elts, 0, 
        llcf->name_values->elts) == NULL) {
            return;
    }

以上是nginx变量编译的全部内容，涵盖了http协议标准变量和非标准变量，从指令参数设计，解析，到nginx核心模块的内置变量导出，非标准变量的提取