---
title: CSSStyleDeclaration.getPropertyValue()
slug: Web/API/CSSStyleDeclaration/getPropertyValue
tags:
- API
- CSSOM
- Method
- Reference
browser-compat: api.CSSStyleDeclaration.getPropertyValue
translation_of: Web/API/CSSStyleDeclaration/getPropertyValue
---
<p>{{ APIRef("CSSOM") }}</p>

<p><strong>CSSStyleDeclaration.getPropertyValue()</strong> メソッドインターフェイスは、指定された CSS プロパティの値を含む {{domxref('DOMString')}} を返します。</p>

<h2 id="Syntax">構文</h2>

<pre
  class="brush: js">var <em>value</em> = <em>style</em>.getPropertyValue(<em>property</em>);</pre>

<h3 id="Parameters">引数</h3>

<ul>
  <li><em><code>property</code></em> は、チェックされるプロパティ名 (ハイフン区切り) を表す {{domxref('DOMString')}} です。</li>
</ul>

<h3 id="Return_value">返値</h3>

<ul>
 <li><code><em>value</em></code> は、プロパティの値を含む {{domxref('DOMString')}} です。設定されていない場合、空文字列を返します。</li>
</ul>

<h2 id="Example">例</h2>

<p>次の JavaScript コードは、 CSS セレクター規則の <code>margin</code> プロパティの値を問い合わせます。</p>

<pre class="brush: js">var declaration = document.styleSheets[0].cssRules[0].style;
var value = declaration.getPropertyValue('margin'); // "1px 2px"
</pre>

<h2 id="Specifications">Specifications</h2>

{{Specifications}}

<h2 id="Browser_compatibility">ブラウザーの互換性</h2>

<p>{{Compat}}</p>