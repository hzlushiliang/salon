# nginx-基数树应用 #

__关键词: nginx radix tree ip地址__

我们都知道，nginx既可以用作伺服器，也可以用作http七层代理，不管用途如何，我们在开发三方内核模块的时候，经常会和客户端的源IP地址打交道，比如说基于客户端的源IP地址线路，对请求做调度，手段可以是httpdns，也可以是IP302，还可以是其他深度定制的方案，这里面需要对IP地址库做一个索引，拿着客户端的IP地址去搜索IP地址库，得到对应的线路值，后续的调度才能谈起

radix二叉基数树非常适合做IP地址或者IP地址段的索引，空间利用率高，索引速度也很快，nginx内核自己实现了radix二叉基数树，核心数据结构有下面两个

    struct ngx_radix_node_s {
        ngx_radix_node_t  *right;
        ngx_radix_node_t  *left;
        ngx_radix_node_t  *parent;
        uintptr_t          value;
    };

    typedef struct {
        ngx_radix_node_t  *root;
        ngx_pool_t        *pool;
        ngx_radix_node_t  *free;
        char              *start;
        size_t             size;
    } ngx_radix_tree_t;

第一个是二叉基数树的节点数据结构，第二个是二叉基数树的全局控制信息

nginx提供了创建，插入，删除，查询四个常用的接口，下面来看看他们的实现

    // 创建
    ngx_radix_tree_t *
    ngx_radix_tree_create(ngx_pool_t *pool, ngx_int_t preallocate)
    {
        tree = ngx_palloc(pool, sizeof(ngx_radix_tree_t));

        tree->root = ngx_radix_alloc(tree);

        // preallocate代表预分配的二叉树深度
        while (preallocate--) {
            mask >>= 1;
            mask |= 0x80000000;

            do {
                ngx_radix32tree_insert(tree, key, mask, NGX_RADIX_NO_VALUE);
                key += inc;
            } while (key);

            inc >>= 1;
        }

        return tree;
    }

创建的时候走nginx内核自带的内存管理接口申请内存，初始化控制结构，该接口带有预分配功能，可以指定预先分配多少层的二叉树节点

    // 插入
    ngx_int_t
    ngx_radix32tree_insert(ngx_radix_tree_t *tree, uint32_t key, uint32_t mask,
        uintptr_t value)
    {
        bit = 0x80000000;
        node = tree->root;
        next = tree->root;

        // 掩码决定了在二叉树的第几层插入数值
        while (bit & mask) {
            // key决定了左右分叉的方向
            if (key & bit) {
                next = node->right;
            } else {
                next = node->left;
            }

            bit >>= 1;
            node = next;
        }

        if (next) {

            node->value = value;
            return NGX_OK;
        }

        while (bit & mask) {
            // 插入新节点
        }

        node->value = value;

        return NGX_OK;
    }

插入节点的时候基于键值和掩码值，定位到二叉树节点，然后赋值，如果二叉树节点尚未生成，创建之

    // 删除
    ngx_int_t
    ngx_radix32tree_delete(ngx_radix_tree_t *tree, uint32_t key, uint32_t mask)
    {
        bit = 0x80000000;
        node = tree->root;

        // 定位二叉树节点
        while (node && (bit & mask)) {
            if (key & bit) {
                node = node->right;

            } else {
                node = node->left;
            }

            bit >>= 1;
        }

        if (node->right || node->left) {
            node->value = NGX_RADIX_NO_VALUE;
            return NGX_OK;
        }

        for ( ;; ) {
            // 向上回溯删除无效节点
        }

        return NGX_OK;
    }

删除节点的时候基于键值和掩码值，定位到二叉树节点，然后清空节点取值，如果节点是叶子节点，则向上回溯删除无效节点

    // 查找
    uintptr_t
    ngx_radix32tree_find(ngx_radix_tree_t *tree, uint32_t key)
    {
        while (node) {
            // 向下遍历的过程中，遇到有效载荷直接记录下来
            if (node->value != NGX_RADIX_NO_VALUE) {
                value = node->value;
            }

            if (key & bit) {
                node = node->right;

            } else {
                node = node->left;
            }

            bit >>= 1;
        }

        return value;
    }

查找节点的时候基于键值，向下遍历，如果遇到有效节点，记录下最新的取值，在基于32位掩码的IP地址查找IP段线路时，该接口十分有效，IP地址的线路都是一段一段的

实际应用的过程中，我们需要对原始IP地址库做一个预处理，将IP地址段映射到一个整数索引，在nginx内核中导入预处理后的地址库，从IP地址段中分离出掩码信息，IP地址段的起始IP作为键值，整数索引作为基数树节点取值，插入到基数树中，后续客户端请求过来，直接拿着客户端IP地址转换后的整型数值去查找基数树，即可定位到客户端所属的线路，这是后续复杂调度的基础工作