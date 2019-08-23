---
title: 使用 underscore.template 扩展 jQuery
date: 2017/12/2 18:2:0
tags: [jQuery,Underscore.js]
categories: [jQuery]
---
最近在接触了一些前端模板框架后，突然觉得以前在 AJAX 操作中拼接组装大量 DOM 对象是多么笨。在了解了 underscore 库的 template 方法后，准备在公司项目中来使用它，但是公司还在用 jQuery，因此准备拿它扩展 jQuery。		
		
<!--more-->
		
		
先上代码，熟悉 underscore 的同学可以略过了。		
```js
/**
 * 使用 underscore.template 扩展 jquery
 * @author nekolr
 */
;(function ($) {
    var escapes = {
        "'":      "'",
        '\\':     '\\',
        '\r':     'r',
        '\n':     'n',
        '\u2028': 'u2028',
        '\u2029': 'u2029'
    }, escapeMap = {
        '&': '&amp;',
        '<': '&lt;',
        '>': '&gt;',
        '"': '&quot;',
        "'": '&#x27;',
        '`': '&#x60;'
    };
    var escapeChar = function(match) {
        return '\\' + escapes[match];
    };
    var createEscaper = function(map) {
        var escaper = function(match) {
            return map[match];
        };
        var source = "(?:&|<|>|\"|'|`)";
        var testRegexp = RegExp(source);
        var replaceRegexp = RegExp(source, 'g');
        return function(string) {
            string = string == null ? '' : '' + string;
            return testRegexp.test(string) ? string.replace(replaceRegexp, escaper) : string;
        };
    };
    var escape = createEscaper(escapeMap);
    $.extend({
        templateSettings: {
            escape: /{{-([\s\S]+?)}}/g,
            interpolate : /{{=([\s\S]+?)}}/g,
            evaluate: /{{([\s\S]+?)}}/g
        },
        escapeHtml: escape,
        template: function (text, settings) {
            var options = $.extend(true, {}, this.templateSettings, settings);
            var matcher = RegExp([options.escape.source,
			options.interpolate.source,
			options.evaluate.source].join('|') + '|$', 'g');
            var index = 0;
            var source = "__p+='";
            text.replace(matcher, function(match, escape, interpolate, evaluate, offset) {
                source += text.slice(index, offset).replace(/\\|'|\r|\n|\u2028|\u2029/g, escapeChar);
                index = offset + match.length;
                if (escape) {
                    source += "'+\n((__t=(" + escape + "))==null?'':$.escapeHtml(__t))+\n'";
                } else if (interpolate) {
                    source += "'+\n((__t=(" + interpolate + "))==null?'':__t)+\n'";
                } else if (evaluate) {
                    source += "';\n" + evaluate + "\n__p+='";
                }
                return match;
            });
            source += "';\n";
            if (!options.variable) source = 'with(obj||{}){\n' + source + '}\n';
            source = "var __t,__p='',__j=Array.prototype.join," +
                "print=function(){__p+=__j.call(arguments,'');};\n" +
                source + 'return __p;\n';
            try {
                var render = new Function(options.variable || 'obj', source);
            } catch (e) {
                e.source = source;
                throw e;
            }
            var template = function(data) {
                return render.call(this, data);
            };
            var argument = options.variable || 'obj';
            template.source = 'function(' + argument + '){\n' + source + '}';
            return template;
        }
    });
})(jQuery);
```
		
简单写个栗子来看看怎么使用吧。		
		
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>test</title>
</head>
<body>
</body>
<script type="text/template" id="tpl">
    {{ for(var i = 0; i < list.length; i++) { }}
        <li>
            <a><img src="resource/img/list.jpg"/></a>
            <h5>
                <a href="http://www.baidu.com?tokenId={{= tokenId }}">{{= list[i].subject }}</a>
            </h5>
            <p>{{= list[i].description }}</p>
            <div class="data">
                <span class="supe-data"><img src="resource/img/data.png">{{= list[i].date }}</span>
                <span><img src="resource/img/zan.png"> 赞 </span>
                <span><img src="resource/img/ly.png"> 回复 </span>
                <span> 已阅 </span>
            </div>
        </li>
    {{ } }}
</script>
<script src="resource/js/jquery-3.0.0.min.js"></script>
<script src="js/template.js"></script>
<script>
    function loadAjax() {
        var tokenId = getQueryString('tokenId');
        $.ajax({
            url: 'http://test.php',
            type: "POST",
            dataType: 'json',
            async: true,
            success: function(msg) {
                var list = msg.list;
                var compiled = $.template($("#tpl").text());
                if(list.length > 0) {
                    var result = compiled({
                        list:list,
                        tokenId: tokenId
                    });
                    $("body").append($(result));
                }
            },
            error: function (jqXHR, textStatus, errorThrown) {
                if (console) {
                    console.log(' 响应状态：[' + jqXHR.status + '], 
					textStatus 状态：[' + textStatus + "], 异常信息：" + errorThrown);
                }
            }
        })
    }
    
    $(function () {
        loadAjax();
    })
</script>
</html>
```