第九章 结构
=============

自然分组的数据被称为结构。我们可以使用Scheme提供的复合数据结构如向量和列表来表示一种“结构”。例如：我们正在处理与树木相关的一组数据。数据（或者叫字段`field`）中的单个元素包括：高度，周长，年龄，树叶形状和树叶颜色共5个字段。这样的数据可以表示为5元向量。这些字段可以利用`vector-ref`访问，或使用`vector-set!`修改。尽管如此，我们仍然不希望记忆向量索引编号与字段的对应关系，这是一个费力且容易出错的事情，尤其是随着时间的流逝，一些字段被加进来，而另一些字段会被删掉。

因此我们使用Scheme的宏`defstruct`去定义一个`结构`，基本上你可以把它当作一种向量，不过它提供了很多方法诸如创建`结构`实例、访问或修改它的字段等等。因此，我们的树结构应这样定义：

```scheme
(defstruct tree height girth age leaf-shape leaf-color)
```

这样它自动生成了一个名为`make-tree`的构造过程，以及每个字段的访问方法，命名为`tree.height`，`tree.girth`等等。构造方法的使用方法如下：

```scheme
(define coconut 
  (make-tree 'height 30
             'leaf-shape 'frond
             'age 5))
```

这个构造函数的参数以成对的形式出现，字段名后面紧跟着其初始值。这些字段能以任意顺序出现，或者不出现——如果字段的值没有定义的话。

访问过程的调用如下所示：

```scheme
(tree.height coconut) =>  30
(tree.leaf-shape coconut) =>  frond
(tree.girth coconut) =>  <undefined>
```

`tree.girth`存取程序返回一个未定义的值，因为我们没有为`coconut`这个`tree`结构指定`girth`的值。

修改过程的调用如下所示：

```scheme
(set!tree.height coconut 40)
(set!tree.girth coconut 10)
```

如果我们现在重新调用访问过程去访问这些字段，我们会得到新的值：

```scheme
(tree.height coconut) =>  40
(tree.girth coconut) =>  10
```

## 9.1 默认初始化

我们可以在定义结构时进行一些初始化的设置，而不是在每个实例中都进行初始化。因此，我们假定`leaf-shape`和`leaf-color`在默认情况下分别为`frond`和`green`。我们可以在调用make-tree时通过显式的初始化来覆盖掉这些默认值，或者在创建一个结构实例后使用上面提到的字段修改过程：

```scheme
(defstruct tree height girth age
                (leaf-shape 'frond)
                (leaf-color 'green))

(define palm (make-tree 'height 60))

(tree.height palm) 
=>  60

(tree.leaf-shape palm) 
=>  frond

(define plantain 
  (make-tree 'height 7
             'leaf-shape 'sheet))

(tree.height plantain) 
=>  7

(tree.leaf-shape plantain) 
=>  sheet

(tree.leaf-color plantain) 
=>  green
```

## 9.2 defstruct定义
宏`defstruct`的定义如下：

```scheme
(define-macro defstruct
  (lambda (s . ff)
    (let ((s-s (symbol->string s)) (n (length ff)))
      (let* ((n+1 (+ n 1))
             (vv (make-vector n+1)))
        (let loop ((i 1) (ff ff))
          (if (<= i n)
            (let ((f (car ff)))
              (vector-set! vv i 
                (if (pair? f) (cadr f) '(if #f #f)))
              (loop (+ i 1) (cdr ff)))))
        (let ((ff (map (lambda (f) (if (pair? f) (car f) f))
                       ff)))
          `(begin
             (define ,(string->symbol 
                       (string-append "make-" s-s))
               (lambda fvfv
                 (let ((st (make-vector ,n+1)) (ff ',ff))
                   (vector-set! st 0 ',s)
                   ,@(let loop ((i 1) (r '()))
                       (if (>= i n+1) r
                           (loop (+ i 1)
                                 (cons `(vector-set! st ,i 
                                          ,(vector-ref vv i))
                                       r))))
                   (let loop ((fvfv fvfv))
                     (if (not (null? fvfv))
                         (begin
                           (vector-set! st 
                               (+ (list-position (car fvfv) ff)
                                  1)
                             (cadr fvfv))
                           (loop (cddr fvfv)))))
                   st)))
             ,@(let loop ((i 1) (procs '()))
                 (if (>= i n+1) procs
                     (loop (+ i 1)
                           (let ((f (symbol->string
                                     (list-ref ff (- i 1)))))
                             (cons
                              `(define ,(string->symbol 
                                         (string-append
                                          s-s "." f))
                                 (lambda (x) (vector-ref x ,i)))
                              (cons
                               `(define ,(string->symbol
                                          (string-append 
                                           "set!" s-s "." f))
                                  (lambda (x v) 
                                    (vector-set! x ,i v)))
                               procs))))))
             (define ,(string->symbol (string-append s-s "?"))
               (lambda (x)
                 (and (vector? x)
                      (eqv? (vector-ref x 0) ',s))))))))))
```
