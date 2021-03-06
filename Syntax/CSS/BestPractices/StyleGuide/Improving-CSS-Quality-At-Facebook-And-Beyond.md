> [原文地址](https://code.facebook.com/posts/879890885467584/improving-css-quality-at-facebook-and-beyond/)



在Facebook里，上千名工程师工作在不同的产品线上，为全世界的用户提供可靠优质的服务，而我们在代码质量管理方面也面临着独一无二的挑战。不仅仅是因为我们面对的是一个庞大的代码基库，还有日渐增加的各种各样的特性，有时候如果你想去重构提高某一个模块，往往会影响到其他很多模块。具体在CSS而言，我们需要处理上千份不停变化的CSS文件。之前我们着力于通过Code Review、代码样式规范以及重构等手段协同工作，而保障代码质量，但是还是会有很多的错误悄悄从眼皮底下溜走，被提交进入到代码库里。我们一直用自建的CSS Linter来检测基本的代码错误与保证一致的编码风格，尽管它基本上已经满足了我们的目标，但还是存在很多的问题，因此我也想在这篇文章里对如何保障CSS的代码质量进行一些讨论。



# Regex is not Enough:之前用的是正则匹配，不咋的啊

老的Linter主要是基于很多个正则表达式对CSS中的语法进行提取，大概是这个样子的：

```

preg_match_all(
 // This pattern matches [attr] selectors with no preceding selector.  
    '/\/\*.*?\*\/|\{[^}]*\}|\s(\[[^\]]+\])/s',
 $data,   
 $matches,   
 PREG_SET_ORDER | PREG_OFFSET_CAPTURE);
 foreach ($matches as $match) {
   if (isset($match[1])) {
     raiseError(...);
   }
```

基本上一个检测规则就需要添加一个专门的匹配规则，非常不好维护，在性能上也有很大的问题。对于每个规则俺们都需要遍历整个文件，性能差得很。

# Abstract Syntax Tree

受够了正则表达式，我们想搞一个更好用的也是更细致的CSS解释器。CSS本身也是一门语言，老把它当做纯文本文件处理也不好，因此我们打算用AST，即抽象语法树的方式构建一个解析器。这种新的处理方式在性能上面有个很不错的提升，譬如我们的代码库中有这么一段CSS代码：

```

{
  display: none:
  background-color: #8B1D3;
  padding: 10px,10px,0,0;
  opacity: 1.0f;
}
```

眼神好的估计才能看出这个代码片段中存在的问题，譬如某个属性名错了、十六进制的颜色代码写错的，分隔符写错了等等。浏览器才不会主动给你报错呢，这样开发者自己也就很难找到错误了。在具体实现上，我们发现[PostCSS](http://postcss.org/) 是个不错的工具，因此我们选择了[Stylelint](http://stylelint.io/)作为我们新的Linter工具，它是基于PostCSS构建的，非常的灵活，社区也不错。

就像JavaScript中的Esprima以及ESLint一样，Stylelint提供了对于完整的AST的访问方式，能够让你根据不同的情况更快速简单的访问具体的代码节点，譬如现在我们的检测规则写成了这个样子：

```

root.walkDecls(node => {
  if (node.prop === 'text-transform' && node.value === 'uppercase') {
    report({
      ...
    });
  }
});
```

我们也可以传入一些基本的函数，譬如`linear-gradient`，就像这个样子：

```

// disallow things like linear-gradient(top, blue, green) w. incorrect first valueroot.walkDecls(node => {
  const parsedValue = styleParser(node.value);
  parsedValue.walk(valueNode => {
    if (valueNode.type === 'function' && valueNode.value === 'linear-gradient') {
      const firstValueInGradient = styleParser.stringify(valueNode.nodes[0]);
      if (disallowedFirstValuesInGradient.indexOf(firstValueInGradient) > -1) {
        report({
          ...
        });
      }
    }
  });
});
```

这样子写出来的检测规则可读性更好，也更好去理解与维护，并且这种方式无论是在怎样的CSS格式化的情况下，以及不管规则和声明放在哪边，都能正常地工作。



# Custom rules:自定义规则

我们默认使用了一些Stylelint内置的规则，譬如[declaration-no-important](https://www.facebook.com/l.php?u=https%3A%2F%2Fgithub.com%2Fstylelint%2Fstylelint%2Ftree%2Fmaster%2Fsrc%2Frules%2Fdeclaration-no-important&h=oAQG1Tctr&s=1),[selector-no-universal](https://github.com/stylelint/stylelint/blob/master/src/rules/selector-no-universal/README.md), 以及 [selector-class-pattern](https://github.com/stylelint/stylelint/tree/master/src/rules/selector-class-pattern)。如何添加自定义规则的方法可以参考[built-in plugin mechanism](http://stylelint.io/developer-guide/plugins/)，而我们使用的譬如：

- 

- slow-css-properties 来告警一些性能较差的属性，譬如opacity或者box-shadow

- filters-with-svg-files 来告警Edge中不支持SVG的过滤

- use-variables来告警那些可以被内置的常量替换的一些变量

- common-properties-whitelist 来检测是否有些误写的其实不存在的属性

- mobile-flexbox 来检测一些不被老版本手机浏览器支持的属性

- text-transform-uppercase 来告警 "text-transform: uppercase"，这个在某些语言表现的不友好



我们也贡献了部分[规则](https://www.facebook.com/l.php?u=https%3A%2F%2Fgithub.com%2Fstylelint%2Fstylelint%2Fpull%2F675&h=GAQF25sgV&s=1) 以及 [ad插件itions](https://www.facebook.com/l.php?u=https%3A%2F%2Fgithub.com%2Fstylelint%2Fstylelint%2Fpull%2F689&h=hAQHu_d3q&s=1) 给Stylelint。



# Automatic replacement:自动替换

我们检测过程中有一个重要的工作就是自动格式化，Linter会在发现某些问题的时候问你是否需要根据规则进行替换，这个功能会节约你大量的手动修改校正的时间。一般来说，我们提交代码之前都会审视下Linter报出的错误，然后去修复这些错误。可惜的是Stylelint并没有内嵌的自动格式化与修复机制，因此我们重写了部分的Stylelint的规则来增加一个自动替换与修复的功能。



![](http://7xi5sw.com1.z0.glb.clouddn.com/13409339_811578745653310_267839981_n.jpg)




# Test all the things

我们老的Linter还有个问题就是没有单元测试，这点就好像代码上线前不进行单元测试一样不靠谱。我们面对的可能是任意格式的处理文本，因此我们也要保证我们的检测规则能够适用于真实有效的环境，这里我们是选择了Jest这个测试框架，Stylelint对它的支持挺好的，然后大概一个单元测试是这个样子：

```

test.ok('div { background-image: linear-gradient( 0deg, blue, green 40%, red ); }', 
  'linear gradient with valid syntax');
test.notOk('a { background: linear-gradient(top, blue, green); }', 
  message, 
    'linear-gradient with invalid syntax');

```

# What‘s next

换一个靠谱的CSS Linter工具只是保证高质量的CSS的代码的第一步，我们还打算添加很多自定义的检测规则来捕获一些常见的错误，保证使用规定的最佳实践以及统一代码约定规范。我们已经在JavaScript的校验中进行了这一工作。

另外对于React社区中存在的CSS-in-JS这种写法，对于CSS Linter也是个不小的挑战，现在的大部分的Linter都是着眼于处理传统的CSS文件，以后会添加对于JSX的处理规范吧。



