---
title: IndexedDB の使用
slug: Web/API/IndexedDB_API/Using_IndexedDB
tags:
  - API
  - Advanced
  - Database
  - Guide
  - IndexedDB
  - Storage
  - Tutorial
  - jsstore
translation_of: Web/API/IndexedDB_API/Using_IndexedDB
---
<p>{{DefaultAPISidebar("IndexedDB")}}</p>

<div class="summary">
<p>IndexedDB は、ユーザーのブラウザー内にデータを永続的に保存する手段です。ネットワークの状態にかかわらず高度な問い合わせ機能を持つウェブアプリケーションを作成できますので、アプリケーションがオンラインとオフラインの両方で動作するようになります。</p>
</div>

<h2 id="About_this_document">このドキュメントについて</h2>

<p>このチュートリアルでは、IndexedDB の非同期 API の使い方を見ていきます。IndexedDB について詳しくない場合は、始めに <a href="/ja/docs/Web/API/IndexedDB_API/Basic_Terminology">IndexedDB の主な特徴と基本用語</a>をお読みください。</p>

<p>IndexedDB API のリファレンスドキュメントとして、<a href="/ja/docs/Web/API/IndexedDB_API">IndexedDB API</a> の記事とそのサブ記事をご覧ください。この記事では、IndexedDB で使用されるオブジェクトの種類と、非同期 API のメソッドについて説明します (同期 API は仕様から削除されました)。</p>

<h2 id="pattern">基本パターン</h2>

<p>IndexedDB で推奨される基本パターンは、以下のようになります。</p>

<ol>
 <li>データベースを開きます。</li>
 <li>データベース内に、オブジェクトストアを作成します。</li>
 <li>データの追加や取り出しといった、データベース操作のトランザクションを開始して、リクエストを行います。</li>
 <li>
  <div>適切な DOM イベントを受け取ることにより、操作が完了するのを待ちます。</div>
 </li>
 <li>
  <div>結果 (リクエストオブジェクトにある) に応じた処理を行います。</div>
 </li>
</ol>

<p>これらの主要な概念を踏まえることで、より具体的な手続きを理解できます。</p>

<h2 id="open">ストアを作成および構築する</h2>

<h3 id="Using_an_experimental_version_of_IndexedDB">実験的なバージョンの IndexedDB を使用する</h3>

<p>まだ接頭辞を使用しているブラウザーでコードのテストを行いたい場合は、以下のコードを使用するとよいでしょう。</p>

<pre class="brush: js">// 以下の行に、テストを行いたい実装の接頭辞を含めてください。
window.indexedDB = window.indexedDB || window.mozIndexedDB || window.webkitIndexedDB || window.msIndexedDB;
// 関数内でない場合は、"var indexedDB = ..." を使用しないでください。
// さらに、window.IDB* オブジェクトへの参照が必要でしょう:
window.IDBTransaction = window.IDBTransaction || window.webkitIDBTransaction || window.msIDBTransaction || {READ_WRITE: "readwrite"}; // この行は、古いブラウザー向けにオブジェクトの定数が必要である場合に限り、必要になります。
window.IDBKeyRange = window.IDBKeyRange || window.webkitIDBKeyRange || window.msIDBKeyRange;
// (Mozilla はこれらのオブジェクトに接頭辞をつけていませんので、window.mozIDB* は不要です)</pre>

<p>接頭辞を使用している実装は不具合がある、未完成、あるいは古い版の仕様に従っている可能性がありますので注意してください。よって、これらを製品版のコードで使用することは推奨しません。対応しているものとして失敗するより、未対応とする方が好ましいでしょう。</p>

<pre class="brush: js">if (!window.indexedDB) {
    window.console.log("このブラウザーは安定版の IndexedDB を対応していません。IndexedDB の機能は利用できません。");
}
</pre>

<h3 id="Opening_a_database">データベースを開く</h3>

<p>プロセス全体は以下のようにして始めます。</p>

<pre class="brush: js">// データベースを開く
var request = window.indexedDB.open("MyTestDatabase", 3);
</pre>

<p>わかりますか? データベースを開くことも他の操作と同様であり、「リクエスト」が必要です。</p>

<p>データベースを開くリクエストは、すぐにはデータベースを開いたりトランザクションを開始したりはしません。<code>open()</code> 関数を呼び出すと、結果 (成功) またはイベントとして扱うエラー値を伴う <a href="/ja/docs/Web/API/IDBOpenDBRequest"><code>IDBOpenDBRequest</code></a> オブジェクトを返します。IndexedDB のほとんどの他の非同期関数も同様であり、結果またはエラーを伴う <a href="/ja/docs/Web/API/IDBRequest"><code>IDBRequest</code></a> オブジェクトを返します。open 関数の結果は <code><a href="/ja/docs/Web/API/IDBDatabase">IDBDatabase</a></code> のインスタンスです。</p>

<p>open メソッドの第 2 引数は、データベースのバージョンです。データベースのバージョンは、データベースのスキーマ、すなわちデータベース内のオブジェクトストアとその構造を決定します。データベースが存在しない場合に <code>open</code> 操作でデータベースが作成されると、<code>onupgradeneeded</code> イベントが発生し、そのイベントハンドラーでデータベースのスキーマを作成することができます。データベースが存在する場合に従来より高いバージョン番号を指定すると、すぐに <code>onupgradeneeded</code> イベントが発生して、そのイベントハンドラーで更新されたスキーマを提供することができます。詳しくは、後ほど<a href="#updating_the_version_of_the_database">データベースのバージョンの更新</a>で説明します。また、{{domxref("IDBFactory.open")}} のリファレンスページもご覧ください。</p>

<div class="warning">
<p><strong>重要</strong>: バージョン番号は <code>unsigned long long</code> 型の数値であり、とても大きい整数にすることができます。また浮動小数点数値は使用できず、使用した場合は <code>upgradeneeded</code> イベントが発生せず、もっとも近い小さな数値に変換されてトランザクションが始まるでしょう。よって、例えばバージョン番号として 2.4 を使用しないでください。<br>
 <code>var request = indexedDB.open("MyTestDatabase", 2.4); // 行ってはいけません。バージョンは 2 に丸められます</code></p>
</div>

<h4 id="Generating_handlers">ハンドラーの生成</h4>

<p>ほぼすべてのリクエスト生成に合わせて始めに行いたいことは、成功およびエラーのハンドラーを生成することでしょう。</p>

<pre class="brush: js">request.onerror = function(event) {
  // request.errorCode に対して行うこと!
};
request.onsuccess = function(event) {
  // request.result に対して行うこと!
};</pre>

<p>2 つの関数 <code>onsuccess()</code> と <code>onerror()</code> のどちらが呼び出されるのでしょう? すべてが成功すると成功イベント (すなわち <code>type</code> プロパティが <code>"success"</code> である DOM イベント) が、<code>request</code> を <code>target</code> として発生します。イベントが発生すると <code>request</code> の <code>onsuccess()</code> 関数が、success イベントを引数として呼び出されます。一方、何らかの問題がある場合はエラーイベント (すなわち <code>type</code> プロパティが <code>"error"</code> である DOM イベント) が <code>request</code> で発生します。これは、エラーイベントを引数として <code>onerror()</code> 関数を呼び出します。</p>

<p>IndexedDB API は必要なエラー処理を最小限にするよう設計されていますので、多くのエラーイベントを見ることはないでしょう (少なくとも、API に慣れていなければ)。しかしデータベースを開く場合は、エラーイベントが発生する一般的な状況があります。もっとも多いであろう問題は、データベースを作成する許可をユーザーがウェブアプリに与えなかったことです。IndexedDB の主要な設計目標のひとつが、オフラインで使用するために大量のデータを保存できるようにすることです。(各ブラウザーでどれだけの量のストレージを持てるかについては、<a href="/ja/docs/Web/API/IndexedDB_API/Browser_storage_limits_and_eviction_criteria#storage_limits" title="IndexedDB#Storage_limits">ストレージの制限</a>をご覧ください)</p>

<p>広告ネットワークやコンピューターを汚染させる悪意のあるウェブサイトをブラウザーが許可したくないことは明らかですので、ブラウザーは ウェブアプリが初めてストレージ用に IndexedDB を開こうとしたときに、ユーザーへプロンプトを表示します。ユーザーはアクセスを許可または拒否できます。またブラウザーのプライバシーモードでの IndexedDB ストレージは、匿名のセッションを閉じるまでの間だけメモリー上に存在します (Firefox のプライベートブラウジングモードや Chrome のシークレットモードのことですが、2021 年 5 月現在の Firefox ではこれが<a href="https://bugzilla.mozilla.org/show_bug.cgi?id=781982">未実装です</a>ので、Firefox のプライベートブラウジングでは IndexedDB をまったく使用できません)。</p>

<p>ユーザーがデータベース作成の要求を許可して、成功コールバックを実行する成功イベントを受け取ったと想定します。次は何を行うのでしょうか? 以下のリクエストは <code>indexedDB.open()</code> の呼び出しを伴って生成されており、<code>request.result</code> は <code>IDBDatabase</code> のインスタンスですので、以降のためにこれを保存したいことは確実です。よって、コードは以下のようになるでしょう。</p>

<pre class="brush: js">var db;
var request = indexedDB.open("MyTestDatabase");
request.onerror = function(event) {
  console.log("なぜ私の ウェブアプリで IndexedDB を使わせてくれないのですか?!");
};
request.onsuccess = function(event) {
  db = event.target.result;
};
</pre>

<h4 id="Handling_Errors">エラーを処理する</h4>

<p>前述のとおり、エラーイベントはバブリングします。エラーイベントはエラーを発生させたリクエストをターゲットにして、さらにトランザクションや最終的にデータベースオブジェクトへバブリングします。すべてのリクエストにエラーハンドラーを追加することを避けたい場合は、代わりに以下のように、ひとつのエラーハンドラーをデータベースオブジェクトに追加することができます。</p>

<pre class="brush: js">db.onerror = function(event) {
  // このデータベースのリクエストに対するすべてのエラー用の
  // 汎用エラーハンドラー!
  console.log("Database error: " + event.target.errorCode);
};
</pre>

<p>データベースを開く際によく発生するエラーのひとつが <code>VER_ERR</code> です。これはディスクに保存されているデータベースのバージョンが、開こうとしているバージョンより<em>大きい</em>ことを表します。これは、必ずエラーハンドラーで処理しなければならないエラーの実例です。</p>

<h3 id="Creating_or_updating_the_version_of_the_database" name="Creating_or_updating_the_version_of_the_database">データベースを作成またはデータベースのバージョンを更新する</h3>

<p><a name="Updating_the_version_of_the_database"></a>新しいデータベースを作成したり既存のデータベースのバージョンを更新したりする (<a href="#opening_a_database">データベースを開く</a>際に、従来より大きなバージョン番号を指定する) と <code>onupgradeneeded</code> イベントが発生して、<code>request.result</code> (すなわち、以下の例の <code>db</code>) に設定した <code>onversionchange</code> イベントハンドラーに <a href="/ja/docs/Web/API/IDBVersionChangeEvent">IDBVersionChangeEvent</a> オブジェクトが渡されます。<code>upgradeneeded</code> イベントのハンドラーでは、このバージョンのデータベースで必要なオブジェクトストアを作成します。</p>

<pre class="brush:js;">// このイベントは最新のブラウザーにのみ実装されています
request.onupgradeneeded = function(event) {
  // IDBDatabase インターフェイスに保存します
  var db = event.target.result;

  // このデータベース用の objectStore を作成します
  var objectStore = db.createObjectStore("name", { keyPath: "myKey" });
};</pre>

<p>この場合、データベースは旧バージョンのデータベース由来のオブジェクトストアをすでに持っていますので、それらのオブジェクトストアを再作成する必要はありません。新しいオブジェクトストアを作成するか、不要になった旧バージョンのオブジェクトストアを削除することだけが必要です。既存のオブジェクトストアを変更しなければならない (例えば <code>keyPath</code> を変更する) 場合は、古いオブジェクトストアを削除してから、新たな設定で再作成しなければならりません。(これはオブジェクトストア内の情報を削除しますので注意してください! この情報を保存しなければならない場合は、データベースをアップグレードする前にデータを読み出して、別の場所に保存してください)</p>

<p>既存の名称を使用してオブジェクトストアを作成しようとする (あるいは、存在しない名称のオブジェクトストアを削除しようとする) と、エラーが発生します。</p>

<p><code>onupgradeneeded</code> イベントから正常に抜けた場合は、データベースを開くリクエストの <code>onsuccess</code> ハンドラーが実行されます。</p>

<h3 id="Structuring_the_database">データベースを構築する</h3>

<p>次に、データベースを構築します。IndexedDB はテーブルではなくオブジェクトストアを使用しており、ひとつのデータベースに複数のオブジェクトストアを含めることができます。値をオブジェクトストアへ保存するたびに、値がキーと関連付けられます。オブジェクトストアで<a href="/ja/docs/Web/API/IndexedDB_API/Basic_Terminology#gloss_keypath">キーパス</a>を使用するか<a href="/ja/docs/Web/API/IndexedDB_API/Basic_Terminology#gloss_keygenerator">キージェネレーター</a>を使用するかに応じて、キーを供給する方法がいくつか存在します。</p>

<p>以下の表で、キーを供給するさまざまな方法を示します。</p>

<table class="standard-table">
 <thead>
  <tr>
   <th scope="col">キーパス (<code>keyPath</code>)</th>
   <th scope="col">キージェネレーター (<code>autoIncrement</code>)</th>
   <th scope="col">説明</th>
  </tr>
 </thead>
 <tbody>
  <tr>
   <td>なし</td>
   <td>なし</td>
   <td>このオブジェクトストアは、数値や文字列といったプリミティブ値を含む、どのような種類の値でも保持できます。新たな値の追加を望むたびに、個別のキー引数を供給しなければなりません。</td>
  </tr>
  <tr>
   <td>あり</td>
   <td>なし</td>
   <td>このオブジェクトストアは、JavaScript オブジェクトのみ保持できます。オブジェクトはキーパスと同じ名称のプロパティを持たなければなりません。</td>
  </tr>
  <tr>
   <td>なし</td>
   <td>あり</td>
   <td>このオブジェクトストアは、どのような種類の値でも保持できます。キーは自動的に生成されます。また、特定のキーを使用したい場合は個別のキー引数を供給できます。</td>
  </tr>
  <tr>
   <td>あり</td>
   <td>あり</td>
   <td>このオブジェクトストアは、JavaScript オブジェクトのみ保持できます。通常はキーが生成されて、オブジェクトでキーパスと同じ名称を持つプロパティに、生成されたキーの値を保存します。ただしそのようなプロパティがすでに存在している場合は、生成された新たなキーではなく、そのプロパティの値をキーとして使用します。</td>
  </tr>
 </tbody>
</table>

<p>オブジェクトストアがプリミティブではなくオブジェクトを保持していれば、オブジェクトストアでインデックスを作成することもできます。インデックスは、オブジェクトのキーではなく保存されたオブジェクトのプロパティの値を使用して、オブジェクトストア内に保存された値を検索することを可能にします。</p>

<p>さらにインデックスには、保存されたデータに単純な制限を強制する機能があります。インデックスを作成する際に unique フラグを設定すると、インデックスのキーパスで同じ値を持つオブジェクトが複数保存されないことを、インデックスが保証します。よって例えば人々の集団の情報を保持するオブジェクトストアがある場合に、同じメールアドレスを持つ人が 2 人存在しないことを保証したい場合は、これを強制するために unique フラグを設定したインデックスを使用するとよいでしょう。</p>

<p>これには混乱するかもしれませんので、シンプルな例で概念を説明するべきでしょう。始めに、例で使用する顧客データをいくつか定義します。</p>

<pre class="brush: js">// 顧客データがどのようなものかを示します
const customerData = [
  { ssn: "444-44-4444", name: "Bill", age: 35, email: "bill@company.com" },
  { ssn: "555-55-5555", name: "Donna", age: 32, email: "donna@home.org" }
];
</pre>

<p>もちろん、誰かの社会保障番号を顧客テーブルの主キーとして使用するべきではないですし (社会保障番号を持っていない人もいます)、年齢の代わりに誕生日を保管することもできますが、これらの不適切な選択は利便性のために無視して先へ進みましょう。</p>

<p>次に、データを保存する IndexedDB を作成するところを見てみましょう。</p>

<pre class="brush: js">const dbName = "the_name";

var request = indexedDB.open(dbName, 2);

request.onerror = function(event) {
  // エラー処理
};
request.onupgradeneeded = function(event) {
  var db = event.target.result;

  // 顧客の情報を保存する objectStore を作成します。
  // "ssn" は一意であることが保証されていますので、キーパスとして使用します。
  // あるいは少なくとも、キックオフミーティングで言われたことです。
  var objectStore = db.createObjectStore("customers", { keyPath: "ssn" });

  // 顧客を名前で検索するためのインデックスを作成します。
  // 重複する可能性がありますので、一意のインデックスとしては使用できません。
  objectStore.createIndex("name", "name", { unique: false });

  // 顧客をメールアドレスで検索するためのインデックスを作成します。2 人の顧客が同じメールアドレスを
  // 使用しないようにしたいので、一意のインデックスを使用します。
  objectStore.createIndex("email", "email", { unique: true });

  // データを追加する前に objectStore の作成を完了させるため、
  // transaction oncomplete を使用します。
  objectStore.transaction.oncomplete = function(event) {
    // 新たに作成した objectStore に値を保存します。
    var customerObjectStore = db.transaction("customers", "readwrite").objectStore("customers");
    customerData.forEach(function(customer) {
      customerObjectStore.add(customer);
    });
  };
};
</pre>

<p>先に示したように、<code>onupgradeneeded</code> はデータベースの構造を変えることができる唯一の場所です。ここではオブジェクトストアの作成および削除や、インデックスの構築および削除が可能です。</p>

<p>オブジェクトストアは <code>createObjectStore()</code> を 1 回呼び出して作成します。このメソッドの引数は、ストアの名前と引数オブジェクトです。引数オブジェクトは省略可能ですが、重要なオプションプロパティを定義したり、作成したいオブジェクトストアの型を改良することができますので、とても重要です。この例では "customers" という名前のオブジェクトストアを要求して、<code>keyPath</code> を定義しています。<code>keyPath</code> は、ストア内で個々のオブジェクトを一意にするプロパティです。この例では、社会保障番号が一意であることが保証されていますので "ssn" にしています。"ssn" は、<code>objectStore</code> に保存するすべてのオブジェクトに与えなければなりません。</p>

<p>また、保存されたオブジェクトの <code>name</code> プロパティを参照する、"name" という名前のインデックスも要求しています。<code>createObjectStore()</code> と同様に <code>createIndex()</code> も、作成したいインデックスの型を改良するための省略可能な引数 <code>options</code> オブジェクトを指定できます。<code>name</code> プロパティを持たないオブジェクトを追加することはできますが、そのオブジェクトは "name" インデックス内に現れません。</p>

<p>以上でオブジェクトストアに保存された顧客オブジェクトを、<code>ssn</code> を使用して直接、またはインデックスを使用して名前をもとにして、取り出すことができます。この仕組みについて詳しくは、<a href="#using_an_index">インデックスを使用する</a> をご覧ください。</p>

<h3 id="Using_a_key_generator">キージェネレーターを使用する</h3>

<p>オブジェクトストアを作成するときに <code>autoIncrement</code> フラグを設定すると、そのオブジェクトストアでキージェネレーターを使用できます。デフォルトで、このフラグは設定されません。</p>

<p>キージェネレーターを使用すると、オブジェクトストアに値を追加するのに応じて自動的にキーが生成されます。オブジェクトストアでキージェネレーターを初めて作成した時点では、キージェネレーターの値が常に 1 になります。基本的に、新たに自動生成されるキーは、前のキーから 1 増加した値になります。データベースのトランザクションが中止されるなど、データベースの操作が取り消された場合を除き、キージェネレーターの現在の値が減少することはありません。従って、オブジェクトストアからレコードを削除したりすべてのレコードをクリアしたりしても、オブジェクトストアのキージェネレーターには影響がありません。</p>

<p>以下のように、キージェネレーターを持つ別のオブジェクトストアを作成できます。</p>

<pre class="brush: js">// indexedDB を開きます。
var request = indexedDB.open(dbName, 3);

request.onupgradeneeded = function (event) {

    var db = event.target.result;

    // autoIncrement フラグに true を設定した、"names" という名前のオブジェクトストアを作成します。
    var objStore = db.createObjectStore("names", { autoIncrement : true });

    // "names" オブジェクトストアはキージェネレーターを持っていますので、値 name のキーは自動的に生成されます。
    // 追加したレコードは以下のようになります:
    // キー : 1 =&gt; 値 : "Bill"
    // キー : 2 =&gt; 値 : "Donna"
    for (var i in customerData) {
        objStore.add(customerData[i].name);
    }
};</pre>

<p>キージェネレーターについて詳しくは、<a href="https://www.w3.org/TR/IndexedDB/#key-generator-concept">"W3C Key Generators"</a> をご覧ください。</p>

<h2 id="Adding_retrieving_and_removing_data">データの追加、読み取り、削除</h2>

<p>新しいデータベースで何かを行えるようにする前に、トランザクションを開始しなければなりません。トランザクションはデータベースオブジェクトから生じており、トランザクションの対象にしたいオブジェクトストアを指定しなければなりません。トランザクションの内部では、データを保持しているオブジェクトストアへのアクセスや、リクエストの実行が可能です。次に、データベースに変更処理を行うのか、あるいはデータベースから読み出すだけかを決めなければなりません。トランザクションは <code>readonly</code>、<code>readwrite</code>、<code>versionchange</code> の 3 つのモードを使用できます。</p>

<p>データベースの「スキーマ」や構造を変更する (オブジェクトストアやインデックスを作成または削除する) には、トランザクションを <code>versionchange</code> モードにしなければなりません。このトランザクションは、<code>version</code> を指定して {{domxref("IDBFactory.open")}} メソッドを呼び出すことによって開きます。</p>

<p>既存のオブジェクトストアからレコードを読み出すには、トランザクションで <code>readonly</code> モードまたは <code>readwrite</code> モードを使用できます。既存のオブジェクトストアに変更処理を行うには、トランザクションを <code>readwrite</code> モードにしなければなりません。このようなトランザクションは {{domxref("IDBDatabase.transaction")}} で開きます。このメソッドの引数は 2 つあり、<code>storeNames</code> (アクセスしたいオブジェクトストアの配列で定義されるスコープ) とトランザクションの <code>mode</code> (<code>readonly</code> または <code>readwrite</code>) です。またこのメソッドは、{{domxref("IDBTransaction.objectStore")}} メソッドを持つトランザクションオブジェクトを返します。<code>objectStore</code> メソッドは、オブジェクトストアにアクセスするために使用できます。デフォルトでは、モードを指定しなければ <code>readonly</code> モードでトランザクションを開きます。</p>

<div class="note">
<p><strong>メモ</strong>: Firefox 40 で、IndexedDB トランザクションはパフォーマンスを向上させるために、永続性の保証を緩和しました ({{Bug("1112702")}} を参照)。以前は <code>readwrite</code> モードのトランザクションで、すべてのデータをディスク上に反映したことが保証された場合に限り {{domxref("IDBTransaction.oncomplete")}} 発生しました。Firefox 40 以降では OS がデータの書き込みを指示した時点で <code>complete</code> が発生しており、実際にはデータがディスク上に反映されていない可能性があります。これにより <code>complete</code> イベントをより早く発生させられますが、データをディスク上に反映する前に OS のクラッシュや電源断が発生するとトランザクション全体を失う危険性が若干あります。このような破壊的な事象はまれですので、ほとんどの利用者は心配する必要がないでしょう。何らかの理由 (例えば、後で再計算できない重要なデータを保存する) で永続性を保証しなければならない場合は、実験的 (非標準) な <code>readwriteflush</code> モード ({{domxref("IDBDatabase.transaction")}} を参照) を使用してトランザクションを生成すると、<code>complete</code> イベントを発生させる前にディスクへの反映を強制させることができます。</p>
</div>

<p>トランザクションで適切なスコープおよびモードを使用すると、データアクセスを高速化できます。ヒントを 2 つ紹介します。</p>

<ul>
 <li>スコープを定義するときは、必要なオブジェクトストアのみ指定します。これにより、同時にスコープが重なり合うことなく、複数のトランザクションを実行できます。</li>
 <li><code>readwrite</code> トランザクションモードは、必要な場合に限り指定します。<code>readonly</code> トランザクションはスコープが重なっても複数同時に実行できますが、<code>readwrite</code> トランザクションはオブジェクトストアに対して 1 個しか実行できません。詳しくは、<a href="/ja/docs/Web/API/IndexedDB_API/Basic_Terminology">IndexedDB の主な特徴と基本用語</a>の記事で <a href="/ja/docs/Web/API/IndexedDB_API/Basic_Terminology#gloss_transaction">トランザクション</a>の定義をご覧ください。</li>
</ul>

<h3 id="Adding_data_to_the_database">データベースにデータを追加する</h3>

<p>データベースを作成したら、書き込みを行いたいと考えるでしょう。これは以下のようにします。</p>

<pre class="brush:js;">var transaction = db.transaction(["customers"], "readwrite");
// メモ: 古い実験的な実装では、"readwrite" の代わりに非推奨の定数 IDBTransaction.READ_WRITE を使用します。
// そのような実装をサポートしたい場合は、以下のように記述します。
// var transaction = db.transaction(["customers"], IDBTransaction.READ_WRITE);</pre>

<p><code>transaction()</code> 関数は引数が 2 つあり (ひとつは省略可能)、トランザクションオブジェクトを返します。第 1 引数は、トランザクションの対象にするオブジェクトストアのリストです。トランザクションですべてのオブジェクトを対象にしたい場合は空の配列を渡すことができますが、仕様書では空の配列に対して InvalidAccessError を生成すべきとされていますので、行わないようにしてください。第 2 引数に何も指定しなければ、読み取り専用のトランザクションになります。書き込みを行いたい場合は、<code>"readwrite"</code> フラグを渡さなければなりません。</p>

<p>以上で、存続期間を理解しなければならないトランザクションができました。トランザクションは、イベントループととても密接に結びついています。トランザクションを作成して、それを使用せずにイベントループに戻ると、トランザクションが非アクティブ状態になります。トランザクションをアクティブにし続ける唯一の方法が、トランザクションでリクエストを行うことです。リクエストが完了すると DOM イベントが発生して、リクエストが成功したと仮定すれば、コールバックの実行中にトランザクションを拡張するもうひとつの機会を得られます。トランザクションを拡張せずにイベントループへ戻ると、トランザクションは非アクティブ状態になります。保留中のリクエストがある限り、トランザクションはアクティブであり続けます。トランザクションの存続期間はごくシンプルですが、慣れるまでには少々時間がかかるでしょう。さらにいくつかの例も、理解する助けになるでしょう。<code>TRANSACTION_INACTIVE_ERR</code> エラーコードを見始めた場合は、何らかの誤りがあるでしょう。</p>

<p>トランザクションは <code>error</code>、<code>abort</code>、<code>complete</code> の 3 種類の DOM イベントを受け取る可能性があります。<code>error</code> イベントがバブリングする方法について話したとおり、トランザクションは、生成したあらゆるリクエストが由来のエラーイベントを受け取ります。より細かいポイントとして、エラー時のデフォルトの動作は、エラーが発生したトランザクションを中止させることです。エラーイベントで始めに <code>stopPropagation()</code> を呼び出して、他の処理を行うようにエラー制御を行わなければ、トランザクション全体がロールバックします。この設計によってエラー制御を考えるよう強いられますが、きめ細かいエラー制御がとても複雑になる場合は、データベースに対して包括的なエラー制御を追加することもできます。エラーイベントを制御しない場合やトランザクションで <code>abort()</code> を呼び出した場合は、トランザクションがロールバックされて、<code>abort</code> イベントが発生します。それ以外の場合は、すべての保留中のリクエストが完了した後に <code>complete</code> イベントが発生します。多くのデータベース操作を行っている場合は、個々のリクエストではなくトランザクションを追跡すると、確実に健全性を促進します。</p>

<p>トランザクションを確保したら、そこからオブジェクトストアを取得しなければならないでしょう。トランザクションは、作成時に指定したオブジェクトストアだけを提供します。そして、必要なデータをすべて追加できます。</p>

<pre class="brush: js">// すべてのデータがデータベースに追加されたときに行う処理
transaction.oncomplete = function(event) {
  console.log("All done!");
};

transaction.onerror = function(event) {
  // エラー制御を忘れずに!
};

var objectStore = transaction.objectStore("customers");
for (var i in customerData) {
  var request = objectStore.add(customerData[i]);
  request.onsuccess = function(event) {
    // event.target.result == customerData[i].ssn;
  };
}</pre>

<p><code>add()</code> を呼び出して生成されたリクエストの <code>result</code> は、追加された値のキーです。よってこのケースでは、オブジェクトストアでキーパスとして <code>ssn</code> プロパティを使用していますので、追加されたオブジェクトの <code>ssn</code> プロパティと等しくなります。<code>add()</code> 関数では、データベース内に同一のキーを持つオブジェクトが存在しないことを要求しますので注意してください。既存の項目を変更しようとする場合や、既存の項目があるかを配慮しない場合は、<a href="#updating_an_entry_in_the_database">データベース内の項目を更新する</a> の章で説明している <code>put()</code> 関数を使用できます。</p>

<h3 id="Removing_data_from_the_database">データベースからのデータの削除</h3>

<p>データの削除もよく似ています。</p>

<pre class="brush: js">var request = db.transaction(["customers"], "readwrite")
                .objectStore("customers")
                .delete("444-44-4444");
request.onsuccess = function(event) {
  // 削除完了!
};</pre>

<h3 id="Getting_data_from_the_database">データベースからのデータの取得</h3>

<p>データベースは情報を持っていますので、いくつかの方法でデータを読み出すことができます。まずは、単純に <code>get()</code> を使用します。以下のように、値を読み出すためにキーを提供しなければなりません。</p>

<pre class="brush: js">var transaction = db.transaction(["customers"]);
var objectStore = transaction.objectStore("customers");
var request = objectStore.get("444-44-4444");
request.onerror = function(event) {
  // エラー処理!
};
request.onsuccess = function(event) {
  // request.result に対して行う処理!
  console.log("Name for SSN 444-44-4444 is " + request.result.name);
};</pre>

<p>"単純に" 読み出すにも多くのコードがあります。データベースレベルでエラー処理を行うとすれば、コードを少々短縮できることを以下に示します。</p>

<pre class="brush: js">db.transaction("customers").objectStore("customers").get("444-44-4444").onsuccess = function(event) {
  console.log("Name for SSN 444-44-4444 is " + event.target.result.name);
};</pre>

<p>どのように動作するかわかりますか? オブジェクトストアが 1 つしかありませんので、トランザクションで必要とするオブジェクトストアのリストを渡さずに、名称を文字列で渡しています。また、データベースから読み出すだけですので、<code>"readwrite"</code> トランザクションは不要です。モードを指定せずに <code>transaction()</code> を呼び出すと、<code>"readonly"</code> トランザクションになります。さらに細かいことですが、実はリクエストオブジェクトを変数に保存していません。DOM イベントはターゲットとしてリクエストを持ちますので、<code>result</code> プロパティを得るためにイベントを使用できます。</p>

<p>トランザクションでスコープやモードを制限することにより、データアクセスを高速化できることに留意してください。ヒントを 2 つ紹介します。</p>

<ul>
 <li><a href="#scope">スコープ</a>を定義するときは、必要なオブジェクトストアのみ指定します。これにより、同時にスコープが重なり合うことなく、複数のトランザクションを実行できます。</li>
 <li>readwrite トランザクションモードは、必要な場合に限り指定します。readonly トランザクションはスコープが重なっても複数同時に実行できますが、readwrite トランザクションはオブジェクトストアに対して 1 個しか実行できません。詳しくは、<a href="/ja/docs/Web/API/IndexedDB_API/Basic_Terminology#gloss_transaction">基本的な概念の記事でトランザクション</a>の定義をご覧ください。</li>
</ul>

<h3 id="Updating_an_entry_in_the_database">データベース内の項目の更新</h3>

<p>読み出したデータを更新して IndexedDB に書き戻す方法は、とてもシンプルです。先ほどのサンプルを多少更新しましょう。</p>

<pre class="brush: js">var objectStore = db.transaction(["customers"], "readwrite").objectStore("customers");
var request = objectStore.get("444-44-4444");
request.onerror = function(event) {
  // エラー処理!
};
request.onsuccess = function(event) {
  // 更新したい、古い値を取得します。
  var data = request.result;

  // オブジェクト内の値を、希望する値に更新します。
  data.age = 42;

  // 更新したオブジェクトを、データベースに書き戻します。
  var requestUpdate = objectStore.put(data);
   requestUpdate.onerror = function(event) {
     // エラーが発生した場合の処理
   };
   requestUpdate.onsuccess = function(event) {
     // 成功 - データを更新しました!
   };
};</pre>

<p>ここでは <code>objectStore</code> を作成して、ssn の値 (<code>444-44-4444</code>) で特定される顧客レコードの取り出しを要求しています。リクエストの結果を変数 (<code>data</code>) に代入して、そのオブジェクトの <code>age</code> プロパティを更新します。そして、顧客レコードを <code>objectStore</code> に書き戻して前の値を上書きする、第 2 のリクエスト (<code>requestUpdate</code>) を作成します。</p>

<div class="note">
<p><strong>メモ</strong>: このケースではデータベースから読み出すだけでなく書き込みも行いたいので、<code>readwrite</code> トランザクションを指定しました。</p>
</div>

<h3 id="Using_a_cursor">カーソルの使用</h3>

<p><code>get()</code> を使用する際は、読み出したいキーがどれかを知っていることが必要です。オブジェクトストア内のすべての値を渡り歩きたい場合は、カーソルを使用できます。以下のようなものです。</p>

<pre class="brush: js">var objectStore = db.transaction("customers").objectStore("customers");

objectStore.openCursor().onsuccess = function(event) {
  var cursor = event.target.result;
  if (cursor) {
    console.log("Name for SSN " + cursor.key + " is " + cursor.value.name);
    cursor.continue();
  }
  else {
    console.log("No more entries!");
  }
};</pre>

<p><code>openCursor()</code> 関数は、引数がいくつかあります。第一に、すぐに取得するキーレンジオブジェクトを使用して、読み出すアイテムの範囲を制限できます。第二に、反復処理を行いたい方向を指定できます。上記の例では、すべてのオブジェクトを昇順方向に反復します。カーソルの成功イベントのコールバックは、やや特殊です。カーソルオブジェクト自体は、リクエストの <code>result</code> です (上記の例では略記法を使用しており、<code>event.target.result</code> になります)。そして実際のキーと値は、カーソルオブジェクトの <code>key</code> プロパティと <code>value</code> プロパティで見つかります。進み続けたい場合は、カーソルで <code>continue()</code> を呼び出さなければなりません。データの終端に達した (または、<code>openCursor()</code> リクエストに一致する項目が存在しない) 場合は成功のコールバックを受け取りますが、<code>result</code> プロパティが <code>undefined</code> になります。</p>

<p>カーソルをよく使用するパターンのひとつが、以下のようにオブジェクトストア内の全データを読み出して、配列に追加することです。</p>

<pre class="brush: js">var customers = [];

objectStore.openCursor().onsuccess = function(event) {
  var cursor = event.target.result;
  if (cursor) {
    customers.push(cursor.value);
    cursor.continue();
  }
  else {
    console.log("Got all customers: " + customers);
  }
};</pre>

<div class="note">
<p><strong>メモ</strong>: それ以外に、このような処理を行うために <code>getAll()</code>  (および <code>getAllKeys()</code>)。を使用することができます。以下のコードは、前出の例とまったく同じことを行います。</p>

<pre class="brush: js">objectStore.getAll().onsuccess = function(event) {
  console.log("Got all customers: " + event.target.result);
};</pre>

<p>これはオブジェクトを横着な方法で作成するため、カーソルの <code>value</code> プロパティの検索に関してパフォーマンスコストが発生します。例えば <code>getAll()</code> を使用するとき、Gecko はすべてのオブジェクトを一度に作成しなければなりません。例えばそれぞれのキーを検索することにのみ関心がある場合は、<code>getAll()</code> よりもカーソルを使用する方がとても効率的です。オブジェクトストア内の全データの配列を得ようとしている場合は、<code>getAll()</code> を使用しましょう。</p>
</div>

<h3 id="Using_an_index">インデックスの使用</h3>

<p>SSN は個人を一意に識別しますので、キーとして SSN を使用して顧客データを保管することは論理的です。(プライバシーの観点でよいアイデアであるかは別の問題であり、この記事の対象外です) 一方、名前で顧客を検索しなければならない場合は、正しいものが見つかるまでデータベース内のすべての SSN に対して反復処理を行わなければなりません。この方法による検索はとても遅いため、代わりにインデックスを使用するとよいでしょう。</p>

<pre class="brush: js">// 最初に、 request.onupgradeneeded の中にインデックスを生成したか確認してください。
// objectStore.createIndex("name", "name");
// まだであれば、 DOMException が発生します。

var index = objectStore.index("name");

index.get("Donna").onsuccess = function(event) {
  console.log("Donna's SSN is " + event.target.result.ssn);
};
</pre>

<p>"name" カーソルは一意ではないので、<code>name</code> に <code>"Donna"</code> が設定されている項目は複数存在する可能性があります。この場合は常に、キーの値がもっとも小さいものを取得します。</p>

<p>指定した <code>name</code> に該当するすべての項目にアクセスしなければならない場合は、カーソルを使用します。インデックス上で、2 種類のカーソルを開くことができます。ノーマルカーソルは、インデックスのプロパティと、オブジェクトストア内のオブジェクトを紐づけます。キーカーソルは、インデックスのプロパティと、オブジェクトストア内にオブジェクトを保存するために使用するキーを紐づけます。これらの違いを以下に示します。</p>

<pre class="brush: js">// 顧客レコードのオブジェクト全体を得るために、ノーマルカーソルを使用します。
index.openCursor().onsuccess = function(event) {
  var cursor = event.target.result;
  if (cursor) {
    // cursor.key は "Bill" のような名前、cursor.value はオブジェクト全体です。
    console.log("Name: " + cursor.key + ", SSN: " + cursor.value.ssn + ", email: " + cursor.value.email);
    cursor.continue();
  }
};

// 顧客レコードのオブジェクトのキーを得るために、キーカーソルを使用します。
index.openKeyCursor().onsuccess = function(event) {
  var cursor = event.target.result;
  if (cursor) {
    // cursor.key は "Bill" のような名前、cursor.value は SSN です。
    // 保存されたオブジェクトの他の部分を直接取得する方法はありません。
    console.log("Name: " + cursor.key + ", SSN: " + cursor.value);
    cursor.continue();
  }
};</pre>

<h3 id="Specifying_the_range_and_direction_of_cursors">カーソルの範囲や方向を指定する</h3>

<p>カーソルで参照する値の範囲を制限したい場合は、<code>IDBKeyRange</code> オブジェクトを使用して、<code>openCursor()</code> または <code>openKeyCursor()</code> の第 1 引数として渡します。ひとつのキーのみ許可するキーレンジ、下限または上限の片方を持つキーレンジ、あるいは下限と上限の両方を持つキーレンジを作成できます。境界は "closed" (すなわち、キーレンジは指定した値を含む) または "open" (すなわち、キーレンジは指定した値を含まない) のどちらにもできます。使い方を以下に示します。</p>

<pre class="brush: js">// "Donna" にのみ一致します。
var singleKeyRange = IDBKeyRange.only("Donna");

// "Bill" より先のすべてに一致します。"Bill" を含みます。
var lowerBoundKeyRange = IDBKeyRange.lowerBound("Bill");

// "Bill" より先のすべてに一致します。ただし "Bill" は含みません。
var lowerBoundOpenKeyRange = IDBKeyRange.lowerBound("Bill", true);

// "Donna" までのすべてに一致します。ただし "Donna" は含みません。
var upperBoundOpenKeyRange = IDBKeyRange.upperBound("Donna", true);

// "Bill" から "Donna" までに一致します。ただし "Donna" は含みません。
var boundKeyRange = IDBKeyRange.bound("Bill", "Donna", false, true);

// いずれかのキーレンジを使用するには、openCursor()/openKeyCursor() の第 1 引数として渡します。
index.openCursor(boundKeyRange).onsuccess = function(event) {
  var cursor = event.target.result;
  if (cursor) {
    // 一致した場合の処理。
    cursor.continue();
  }
};</pre>

<p>昇順 (すべてのカーソルのデフォルトの方向) ではなく、降順に反復処理を行いたい場合があるかもしれません。方向の切り替えは、<code>openCursor()</code> の第 2 引数に <code>prev</code> を渡すことで実現します。</p>

<pre class="brush: js">objectStore.openCursor(boundKeyRange, "prev").onsuccess = function(event) {
  var cursor = event.target.result;
  if (cursor) {
    // 項目に対して行う処理
    cursor.continue();
  }
};</pre>

<p>方向を変えたいだけで表示する結果は制限しない場合は、第 1 引数に null を渡します。</p>

<pre class="brush: js">objectStore.openCursor(null, "prev").onsuccess = function(event) {
  var cursor = event.target.result;
  if (cursor) {
    // 項目に対して行う処理
    cursor.continue();
  }
};</pre>

<p>"name" インデックスは一意ではありませんので、<code>name</code> が同じ項目が複数存在する可能性があります。キーは常に一意でなければならないため、オブジェクトストアでこのような状況は発生できないことに注意してください。インデックスに対して反復処理を行う際に重複を取り除きたい場合は、方向の引数に <code>nextunique</code> (逆向きであれば <code>prevunique</code>) を指定します。<code>nextunique</code> または <code>prevunique</code> を使用すると、常にキーが最小の項目が返ります。</p>

<pre class="brush: js">index.openKeyCursor(null, "nextunique").onsuccess = function(event) {
  var cursor = event.target.result;
  if (cursor) {
    // 項目に対して行う処理
    cursor.continue();
  }
};</pre>

<p>有効な方向の引数については、"<a href="/ja/docs/Web/API/IDBCursor#constants">IDBCursor Constants</a>" をご覧ください。</p>

<h2 id="Version_changes_while_a_web_app_is_open_in_another_tab">ウェブアプリが別のタブで開かれているときにバージョンを変更する</h2>

<p>データベースのバージョン変更が必要である場合に ウェブアプリでそのようなことを行うときは、ユーザーが古いバージョンの ウェブアプリをタブで開いている場合に別のタブで新しいバージョンのアプリを読み込んだときに発生することを考慮しなければなりません。データベースの実際のバージョンより大きなバージョンを指定して <code>open()</code> を呼び出すときは、データベースに変更を施す前に、他にデータベースを開いているものが明示的に要求を認めなければなりません (それらを閉じるか再読み込みするまで、<code>onblocked</code> イベントが発生します)。使い方を以下に示します。</p>

<pre class="brush: js">var openReq = mozIndexedDB.open("MyTestDatabase", 2);

openReq.onblocked = function(event) {
  // 他のタブがデータベースを読み込んでいる場合は、処理を進める前に
  // それらを閉じなければなりません。
  console.log("このサイトを開いている他のタブをすべて閉じてください!");
};

openReq.onupgradeneeded = function(event) {
  // 他のデータベースはすべて閉じられました。すべての処理を行います。
  db.createObjectStore(/* ... */);
  useDatabase(db);
};

openReq.onsuccess = function(event) {
  var db = event.target.result;
  useDatabase(db);
  return;
};

function useDatabase(db) {
  // 別のページがバージョン変更を求めた場合に、通知されるようにするためのハンドラーを追加するようにしてください。
  // データベースを閉じなければなりません。データベースを閉じると、別のページがデータベースをアップグレードできます。
  // これを行わなければ、ユーザーがタブを閉じるまでデータベースはアップグレードされません。
  db.onversionchange = function(event) {
    db.close();
    console.log("新しいバージョンのページが使用可能になりました。再読み込みしてください!");
  };

  // データベースを使用する処理
}
</pre>

<p>すでに開かれているアプリが新たにデータベースを開こうとするコードを開始したが、古いバージョンのデータベースを使用している状況に対処するため、<code>VersionError</code> エラーもリッスンしましょう。</p>

<h2 id="Security">セキュリティ</h2>

<p>IndexedDB は同一生成元の原則を使用します。すなわち、ストアとサイトの生成元 (通常、サイトのドメインまたはサブドメイン) を紐づけますので、他の生成元からアクセスすることはできません。</p>

<p>サードパーティの window コンテンツ (例えば {{htmlelement("iframe")}} のコンテンツ) は、ブラウザーが<a href="https://support.mozilla.org/ja/kb/disable-third-party-cookies">サードパーティ Cookie を禁止していない</a>限り、自身が埋め込まれている生成元の IndexedDB ストアにアクセスできます ({{bug("1147821")}} をご覧ください)。</p>

<h2 id="Warning_About_Browser_Shutdown">ブラウザーの終了に関する警告</h2>

<p>ブラウザーを終了するとき (例えばユーザーが「終了」や「閉じる」ボタンをクリックしたとき)、データベースを含むディスクは予期せず削除されたり、データベースストアへのパーミッションが失われたり、次のことが起きたりします。</p>

<ol>
 <li>影響するデータベース (あるいは、ブラウザーを終了する場合はすべての開いているデータベース) の各トランザクションは <code>AbortError</code> とともに中断されます。この効果は各トランザクションで {{domxref("IDBTransaction.abort()")}} が呼ばれたのと同等です。</li>
 <li>すべてのトランザクションが完了していたら、データベース接続は閉じられます。</li>
 <li>最後に、データベース接続を表す {{domxref("IDBDatabase")}} オブジェクトは {{event("close")}} イベントを受け取ります。{{domxref("IDBDatabase.onclose")}} イベントハンドラーを使ってこのイベントを待ち受けできます。その結果、データベースが予期せず閉じられたことがわかります。</li>
</ol>

<p>上記の挙動は新しく、ブラウザーの Firefox 50、Google Chrome 31 以降 (おおよそ) のリリースで利用できます。</p>

<p>このバージョンのブラウザーの前は、トランザクションは暗黙裡に中断され、{{event("close")}} イベントが発行されず、予期せぬデータベースの停止を検出する方法はありませんでした。</p>

<p>ユーザーはいつでもブラウザーを終了することができますので、特定のトランザクションが完了することをあてにしたり、完了しなかったことを知ったりすることはできません。この動作が暗示することがいくつかあります。</p>

<p>第一に、データベースであらゆるトランザクションが終了したときに、常に一貫性がある状態を保つように注意するべきです。例えば、ユーザーが編集可能な項目のリストを保存する IndexedDB を使用していると想定します。オブジェクトストアを消去してから新たなリストを書き込むことにより、編集後のリストを保存します。あるトランザクションでオブジェクトストアを消去して、別のトランザクションで新たなリストを書き込むとすれば、消去した後かつ書き込む前にブラウザーが閉じられる危険性があり、その場合は空のデータベースが残ります。これを避けるために、消去と書き込みをひとつのトランザクションに結合しましょう。</p>

<p>第二に、データベースのトランザクションと unload イベントを紐づけるべきではありません。ブラウザーを閉じることで unload イベントが発生した場合、unload イベントハンドラーで作成したトランザクションは完了しません。ブラウザーのセッションにわたって情報を管理するための直感的な方法は、ブラウザー (または特定のページ) を開いたときに情報を読み込んで、ユーザーとブラウザーとの対話に応じて更新して、ブラウザー (またはページ) を閉じるときに保存する流れです。しかし、これは動作しないでしょう。データベースのトランザクションは unload イベントハンドラーで作成されますが、これらは非同期処理ですので、実行できるようになる前に中止されるでしょう。</p>

<p>実は通常のブラウザー終了であっても、IndexedDB のトランザクションが完了するよう保証する手段はありません。{{bug(870645)}} をご覧ください。通常の終了通知の回避策として、トランザクションの状況を追跡して、アンロード時にトランザクションが完了していないことをユーザーに警告するための <code>beforeunload</code> イベントを追加するとよいでしょう。</p>

<p>少なくとも中止通知と{{domxref("IDBDatabase.onclose")}}を追加することで、いつ起こったのかがわかります。</p>

<h2 id="Locale-aware_sorting">ロケールを意識した並べ替え</h2>

<p>Mozilla は Firefox 43 以降に、IndexedDB のデータでロケールを意識した並べ替えを行う機能を実装しました。デフォルトでは、IndexedDB は文字列の並べ替えで国際化にまったく対処せず、すべてが英語のテキストであるかのように並べ替えられます。例えば b、á、z、a は以下のように並べ替えられます。</p>

<ul>
 <li>a</li>
 <li>b</li>
 <li>z</li>
 <li>á</li>
</ul>

<p>これは明らかに、ユーザーが望むデータの並べ替えではありません。例えば Aaron と Áaron は、連絡先一覧で隣り合うべきです。従って適切な国際化並べ替えを実現するには、データセット全体をメモリーに読み込んで、クライアントサイド JavaScript で並べ替えを実行しなければならず、非効率的です。</p>

<p>この新機能は、開発者が {{domxref("IDBObjectStore.createIndex()")}} を使用してインデックスを作成する際にロケールを指定できるようにします (引数を確認してください)。データセットに対して反復処理を行うためにカーソルを使用するときに、ロケールを意識した並べ替えを行いたい場合は、特化した {{domxref("IDBLocaleAwareKeyRange")}} を使用できます。</p>

<p>また {{domxref("IDBIndex")}} には、ロケールが指定されているか、およびどのロケールが指定されているかを特定するために追加された新たなプロパティがあります。<code>locale</code> (指定されたロケール、または未指定であれば null を返します) と <code>isAutoLocale</code> (プラットフォームの既定のロケールを使用する自動ロケールでインデックスが作成されていれば <code>true</code>、そうでなければ <code>false</code> を返します) です。</p>

<div class="note">
<p><strong>メモ</strong>: 現在、この機能はフラグで隠されています。有効化して実験するには、<a>about:config</a> に移動して <code>dom.indexedDB.experimental</code> を有効化してください。</p>
</div>

<h2 id="Full_IndexedDB_example">包括的な IndexedDB のサンプル</h2>

<h3 id="HTML_Content">HTML コンテンツ</h3>

<pre class="brush: html">&lt;script type="text/javascript" src="https://ajax.googleapis.com/ajax/libs/jquery/1.8.3/jquery.min.js"&gt;&lt;/script&gt;

    &lt;h1&gt;IndexedDB Demo: storing blobs, e-publication example&lt;/h1&gt;
    &lt;div class="note"&gt;
      &lt;p&gt;
        Works and tested with:
      &lt;/p&gt;
      &lt;div id="compat"&gt;
      &lt;/div&gt;
    &lt;/div&gt;

    &lt;div id="msg"&gt;
    &lt;/div&gt;

    &lt;form id="register-form"&gt;
      &lt;table&gt;
        &lt;tbody&gt;
          &lt;tr&gt;
            &lt;td&gt;
              &lt;label for="pub-title" class="required"&gt;
                Title:
              &lt;/label&gt;
            &lt;/td&gt;
            &lt;td&gt;
              &lt;input type="text" id="pub-title" name="pub-title" /&gt;
            &lt;/td&gt;
          &lt;/tr&gt;
          &lt;tr&gt;
            &lt;td&gt;
              &lt;label for="pub-biblioid" class="required"&gt;
                Bibliographic ID:&lt;br/&gt;
                &lt;span class="note"&gt;(ISBN, ISSN, etc.)&lt;/span&gt;
              &lt;/label&gt;
            &lt;/td&gt;
            &lt;td&gt;
              &lt;input type="text" id="pub-biblioid" name="pub-biblioid"/&gt;
            &lt;/td&gt;
          &lt;/tr&gt;
          &lt;tr&gt;
            &lt;td&gt;
              &lt;label for="pub-year"&gt;
                Year:
              &lt;/label&gt;
            &lt;/td&gt;
            &lt;td&gt;
              &lt;input type="number" id="pub-year" name="pub-year" /&gt;
            &lt;/td&gt;
          &lt;/tr&gt;
        &lt;/tbody&gt;
        &lt;tbody&gt;
          &lt;tr&gt;
            &lt;td&gt;
              &lt;label for="pub-file"&gt;
                File image:
              &lt;/label&gt;
            &lt;/td&gt;
            &lt;td&gt;
              &lt;input type="file" id="pub-file"/&gt;
            &lt;/td&gt;
          &lt;/tr&gt;
          &lt;tr&gt;
            &lt;td&gt;
              &lt;label for="pub-file-url"&gt;
                Online-file image URL:&lt;br/&gt;
                &lt;span class="note"&gt;(same origin URL)&lt;/span&gt;
              &lt;/label&gt;
            &lt;/td&gt;
            &lt;td&gt;
              &lt;input type="text" id="pub-file-url" name="pub-file-url"/&gt;
            &lt;/td&gt;
          &lt;/tr&gt;
        &lt;/tbody&gt;
      &lt;/table&gt;

      &lt;div class="button-pane"&gt;
        &lt;input type="button" id="add-button" value="Add Publication" /&gt;
        &lt;input type="reset" id="register-form-reset"/&gt;
      &lt;/div&gt;
    &lt;/form&gt;

    &lt;form id="delete-form"&gt;
      &lt;table&gt;
        &lt;tbody&gt;
          &lt;tr&gt;
            &lt;td&gt;
              &lt;label for="pub-biblioid-to-delete"&gt;
                Bibliographic ID:&lt;br/&gt;
                &lt;span class="note"&gt;(ISBN, ISSN, etc.)&lt;/span&gt;
              &lt;/label&gt;
            &lt;/td&gt;
            &lt;td&gt;
              &lt;input type="text" id="pub-biblioid-to-delete"
                     name="pub-biblioid-to-delete" /&gt;
           &lt;/td&gt;
          &lt;/tr&gt;
          &lt;tr&gt;
            &lt;td&gt;
              &lt;label for="key-to-delete"&gt;
                Key:&lt;br/&gt;
                &lt;span class="note"&gt;(例えば、 1, 2, 3, etc.)&lt;/span&gt;
              &lt;/label&gt;
            &lt;/td&gt;
            &lt;td&gt;
              &lt;input type="text" id="key-to-delete"
                     name="key-to-delete" /&gt;
            &lt;/td&gt;
          &lt;/tr&gt;
        &lt;/tbody&gt;
      &lt;/table&gt;
      &lt;div class="button-pane"&gt;
        &lt;input type="button" id="delete-button" value="Delete Publication" /&gt;
        &lt;input type="button" id="clear-store-button"
               value="Clear the whole store" class="destructive" /&gt;
      &lt;/div&gt;
    &lt;/form&gt;

    &lt;form id="search-form"&gt;
      &lt;div class="button-pane"&gt;
        &lt;input type="button" id="search-list-button"
               value="List database content" /&gt;
      &lt;/div&gt;
    &lt;/form&gt;

    &lt;div&gt;
      &lt;div id="pub-msg"&gt;
      &lt;/div&gt;
      &lt;div id="pub-viewer"&gt;
      &lt;/div&gt;
      &lt;ul id="pub-list"&gt;
      &lt;/ul&gt;
    &lt;/div&gt;
</pre>

<h3 id="CSS_Content">CSS コンテンツ</h3>

<pre class="brush: css">body {
  font-size: 0.8em;
  font-family: Sans-Serif;
}

form {
  background-color: #cccccc;
  border-radius: 0.3em;
  display: inline-block;
  margin-bottom: 0.5em;
  padding: 1em;
}

table {
  border-collapse: collapse;
}

input {
  padding: 0.3em;
  border-color: #cccccc;
  border-radius: 0.3em;
}

.required:after {
  content: "*";
  color: red;
}

.button-pane {
  margin-top: 1em;
}

#pub-viewer {
  float: right;
  width: 48%;
  height: 20em;
  border: solid #d092ff 0.1em;
}
#pub-viewer iframe {
  width: 100%;
  height: 100%;
}

#pub-list {
  width: 46%;
  background-color: #eeeeee;
  border-radius: 0.3em;
}
#pub-list li {
  padding-top: 0.5em;
  padding-bottom: 0.5em;
  padding-right: 0.5em;
}

#msg {
  margin-bottom: 1em;
}

.action-success {
  padding: 0.5em;
  color: #00d21e;
  background-color: #eeeeee;
  border-radius: 0.2em;
}

.action-failure {
  padding: 0.5em;
  color: #ff1408;
  background-color: #eeeeee;
  border-radius: 0.2em;
}

.note {
  font-size: smaller;
}

.destructive {
  background-color: orange;
}
.destructive:hover {
  background-color: #ff8000;
}
.destructive:active {
  background-color: red;
}
</pre>

<h3 id="JavaScript_Content">JavaScript コンテンツ</h3>

<pre class="brush: js">(function () {
  var COMPAT_ENVS = [
    ['Firefox', "&gt;= 16.0"],
    ['Google Chrome',
     "&gt;= 24.0 (you may need to get Google Chrome Canary), NO Blob storage support"]
  ];
  var compat = $('#compat');
  compat.empty();
  compat.append('&lt;ul id="compat-list"&gt;&lt;/ul&gt;');
  COMPAT_ENVS.forEach(function(val, idx, array) {
    $('#compat-list').append('&lt;li&gt;' + val[0] + ': ' + val[1] + '&lt;/li&gt;');
  });

  const DB_NAME = 'mdn-demo-indexeddb-epublications';
  const DB_VERSION = 1; // この値は long long を使用します (float ではありません)
  const DB_STORE_NAME = 'publications';

  var db;

  // ビューの無駄な再読み込みを避けるため、どのビューが表示されているかを追跡するために使用します
  var current_view_pub_key;

  function openDb() {
    console.log("openDb ...");
    var req = indexedDB.open(DB_NAME, DB_VERSION);
    req.onsuccess = function (evt) {
      // db = req.result; と同等
      db = this.result;
      console.log("openDb DONE");
    };
    req.onerror = function (evt) {
      console.error("openDb:", evt.target.errorCode);
    };

    req.onupgradeneeded = function (evt) {
      console.log("openDb.onupgradeneeded");
      var store = evt.currentTarget.result.createObjectStore(
        DB_STORE_NAME, { keyPath: 'id', autoIncrement: true });

      store.createIndex('biblioid', 'biblioid', { unique: true });
      store.createIndex('title', 'title', { unique: false });
      store.createIndex('year', 'year', { unique: false });
    };
  }

  /**
   * @param {string} store_name
   * @param {string} mode "readonly" または "readwrite"
   */
  function getObjectStore(store_name, mode) {
    var tx = db.transaction(store_name, mode);
    return tx.objectStore(store_name);
  }

  function clearObjectStore() {
    var store = getObjectStore(DB_STORE_NAME, 'readwrite');
    var req = store.clear();
    req.onsuccess = function(evt) {
      displayActionSuccess("Store cleared");
      displayPubList(store);
    };
    req.onerror = function (evt) {
      console.error("clearObjectStore:", evt.target.errorCode);
      displayActionFailure(this.error);
    };
  }

  function getBlob(key, store, success_callback) {
    var req = store.get(key);
    req.onsuccess = function(evt) {
      var value = evt.target.result;
      if (value)
        success_callback(value.blob);
    };
  }

  /**
   * @param {IDBObjectStore=} store
   */
  function displayPubList(store) {
    console.log("displayPubList");

    if (typeof store == 'undefined')
      store = getObjectStore(DB_STORE_NAME, 'readonly');

    var pub_msg = $('#pub-msg');
    pub_msg.empty();
    var pub_list = $('#pub-list');
    pub_list.empty();
    // 以前のコンテンツを表示しないようにするため、iframe をリセットします
    newViewerFrame();

    var req;
    req = store.count();
    // リクエストは、トランザクションに対して作成された順で実行され、
    // 結果も同じ順序で返されます。
    // よって、以下の計数は実際の pub のリストより前に表示されるかもしれません
    // (このケースでは、アルゴリズム的に重要ではありません)。
    req.onsuccess = function(evt) {
      pub_msg.append('&lt;p&gt;There are &lt;strong&gt;' + evt.target.result +
                     '&lt;/strong&gt; record(s) in the object store.&lt;/p&gt;');
    };
    req.onerror = function(evt) {
      console.error("add error", this.error);
      displayActionFailure(this.error);
    };

    var i = 0;
    req = store.openCursor();
    req.onsuccess = function(evt) {
      var cursor = evt.target.result;

      // カーソルが何かを指している場合に、データを要求します
      if (cursor) {
        console.log("displayPubList cursor:", cursor);
        req = store.get(cursor.key);
        req.onsuccess = function (evt) {
          var value = evt.target.result;
          var list_item = $('&lt;li&gt;' +
                            '[' + cursor.key + '] ' +
                            '(biblioid: ' + value.biblioid + ') ' +
                            value.title +
                            '&lt;/li&gt;');
          if (value.year != null)
            list_item.append(' - ' + value.year);

          if (value.hasOwnProperty('blob') &amp;&amp;
              typeof value.blob != 'undefined') {
            var link = $('&lt;a href="' + cursor.key + '"&gt;File&lt;/a&gt;');
            link.on('click', function() { return false; });
            link.on('mouseenter', function(evt) {
                      setInViewer(evt.target.getAttribute('href')); });
            list_item.append(' / ');
            list_item.append(link);
          } else {
            list_item.append(" / No attached file");
          }
          pub_list.append(list_item);
        };

        // ストア内の次のオブジェクトに移動する
        cursor.continue();

        // このカウンターは、個別の ID を作成するためだけに使用する
        i++;
      } else {
        console.log("No more entries");
      }
    };
  }

  function newViewerFrame() {
    var viewer = $('#pub-viewer');
    viewer.empty();
    var iframe = $('&lt;iframe /&gt;');
    viewer.append(iframe);
    return iframe;
  }

  function setInViewer(key) {
    console.log("setInViewer:", arguments);
    key = Number(key);
    if (key == current_view_pub_key)
      return;

    current_view_pub_key = key;

    var store = getObjectStore(DB_STORE_NAME, 'readonly');
    getBlob(key, store, function(blob) {
      console.log("setInViewer blob:", blob);
      var iframe = newViewerFrame();

      // 直接ダウンロードするという意味で、blob に
      // 直接リンクを設定することはできません。
      if (blob.type == 'text/html') {
        var reader = new FileReader();
        reader.onload = (function(evt) {
          var html = evt.target.result;
          iframe.load(function() {
            $(this).contents().find('html').html(html);
          });
        });
        reader.readAsText(blob);
      } else if (blob.type.indexOf('image/') == 0) {
        iframe.load(function() {
          var img_id = 'image-' + key;
          var img = $('&lt;img id="' + img_id + '"/&gt;');
          $(this).contents().find('body').html(img);
          var obj_url = window.URL.createObjectURL(blob);
          $(this).contents().find('#' + img_id).attr('src', obj_url);
          window.URL.revokeObjectURL(obj_url);
        });
      } else if (blob.type == 'application/pdf') {
        $('*').css('cursor', 'wait');
        var obj_url = window.URL.createObjectURL(blob);
        iframe.load(function() {
          $('*').css('cursor', 'auto');
        });
        iframe.attr('src', obj_url);
        window.URL.revokeObjectURL(obj_url);
      } else {
        iframe.load(function() {
          $(this).contents().find('body').html("No view available");
        });
      }

    });
  }

  /**
   * @param {string} biblioid
   * @param {string} title
   * @param {number} year
   * @param {string} url ダウンロードしてローカルの IndexedDB データベースに保存する
   *   画像の URL。この URL の背後にあるリソースは、"同一生成元ポリシー" に従います。
   *   よって、この方法を動作させるために URL は、このコードを配置する
   *  ウェブサイト/アプリと同一生成元であることが必要です。
   */
  function addPublicationFromUrl(biblioid, title, year, url) {
    console.log("addPublicationFromUrl:", arguments);

    var xhr = new XMLHttpRequest();
    xhr.open('GET', url, true);
    // 希望する responseType を "blob" に設定
    // http://www.w3.org/TR/XMLHttpRequest2/#the-response-attribute
    xhr.responseType = 'blob';
    xhr.onload = function (evt) {
      if (xhr.status == 200) {
        console.log("Blob retrieved");
        var blob = xhr.response;
        console.log("Blob:", blob);
        addPublication(biblioid, title, year, blob);
      } else {
        console.error("addPublicationFromUrl error:",
        xhr.responseText, xhr.status);
      }
    };
    xhr.send();

    // jQuery 1.8.3 では新しい "blob" responseType を扱わないため、
    // ここでは jQuery を使用できません。
    // http://bugs.jquery.com/ticket/11461
    // http://bugs.jquery.com/ticket/7248
    // $.ajax({
    //   url: url,
    //   type: 'GET',
    //   xhrFields: { responseType: 'blob' },
    //   success: function(data, textStatus, jqXHR) {
    //     console.log("Blob retrieved");
    //     console.log("Blob:", data);
    //     // addPublication(biblioid, title, year, data);
    //   },
    //   error: function(jqXHR, textStatus, errorThrown) {
    //     console.error(errorThrown);
    //     displayActionFailure("Error during blob retrieval");
    //   }
    // });
  }

  /**
   * @param {string} biblioid
   * @param {string} title
   * @param {number} year
   * @param {Blob=} blob
   */
  function addPublication(biblioid, title, year, blob) {
    console.log("addPublication arguments:", arguments);
    var obj = { biblioid: biblioid, title: title, year: year };
    if (typeof blob != 'undefined')
      obj.blob = blob;

    var store = getObjectStore(DB_STORE_NAME, 'readwrite');
    var req;
    try {
      req = store.add(obj);
    } catch (e) {
      if (e.name == 'DataCloneError')
        displayActionFailure("This engine doesn't know how to clone a Blob, " +
                             "use Firefox");
      throw e;
    }
    req.onsuccess = function (evt) {
      console.log("Insertion in DB successful");
      displayActionSuccess();
      displayPubList(store);
    };
    req.onerror = function() {
      console.error("addPublication error", this.error);
      displayActionFailure(this.error);
    };
  }

  /**
   * @param {string} biblioid
   */
  function deletePublicationFromBib(biblioid) {
    console.log("deletePublication:", arguments);
    var store = getObjectStore(DB_STORE_NAME, 'readwrite');
    var req = store.index('biblioid');
    req.get(biblioid).onsuccess = function(evt) {
      if (typeof evt.target.result == 'undefined') {
        displayActionFailure("No matching record found");
        return;
      }
      deletePublication(evt.target.result.id, store);
    };
    req.onerror = function (evt) {
      console.error("deletePublicationFromBib:", evt.target.errorCode);
    };
  }

  /**
   * @param {number} key
   * @param {IDBObjectStore=} store
   */
  function deletePublication(key, store) {
    console.log("deletePublication:", arguments);

    if (typeof store == 'undefined')
      store = getObjectStore(DB_STORE_NAME, 'readwrite');

    // 仕様書 http://www.w3.org/TR/IndexedDB/#object-store-deletion-operation によれば
    // Object Store Deletion Operation アルゴリズムの結果は undefined であり、
    // あるレコードが実際に削除されたかを、リクエストの結果を確認して
    // 知ることはできません。
    var req = store.get(key);
    req.onsuccess = function(evt) {
      var record = evt.target.result;
      console.log("record:", record);
      if (typeof record == 'undefined') {
        displayActionFailure("No matching record found");
        return;
      }
      // 警告: 削除するには、作成時に使用したものとまったく同じキーを使用しなければ
      // なりません。作成時のキーが数値であった場合は、削除時も数値でなければ
      // なりません。
      var deleteReq = store.delete(key);
      deleteReq.onsuccess = function(evt) {
        console.log("evt:", evt);
        console.log("evt.target:", evt.target);
        console.log("evt.target.result:", evt.target.result);
        console.log("delete successful");
        displayActionSuccess("Deletion successful");
        displayPubList(store);
      };
      deleteReq.onerror = function (evt) {
        console.error("deletePublication:", evt.target.errorCode);
      };
    };
    req.onerror = function (evt) {
      console.error("deletePublication:", evt.target.errorCode);
    };
  }

  function displayActionSuccess(msg) {
    msg = typeof msg != 'undefined' ? "Success: " + msg : "Success";
    $('#msg').html('&lt;span class="action-success"&gt;' + msg + '&lt;/span&gt;');
  }
  function displayActionFailure(msg) {
    msg = typeof msg != 'undefined' ? "Failure: " + msg : "Failure";
    $('#msg').html('&lt;span class="action-failure"&gt;' + msg + '&lt;/span&gt;');
  }
  function resetActionStatus() {
    console.log("resetActionStatus ...");
    $('#msg').empty();
    console.log("resetActionStatus DONE");
  }

  function addEventListeners() {
    console.log("addEventListeners");

    $('#register-form-reset').click(function(evt) {
      resetActionStatus();
    });

    $('#add-button').click(function(evt) {
      console.log("add ...");
      var title = $('#pub-title').val();
      var biblioid = $('#pub-biblioid').val();
      if (!title || !biblioid) {
        displayActionFailure("Required field(s) missing");
        return;
      }
      var year = $('#pub-year').val();
      if (year != '') {
        // EcmaScript 6 エンジンでは use Number.isInteger を使用するとよい
        if (isNaN(year))  {
          displayActionFailure("Invalid year");
          return;
        }
        year = Number(year);
      } else {
        year = null;
      }

      var file_input = $('#pub-file');
      var selected_file = file_input.get(0).files[0];
      console.log("selected_file:", selected_file);
      // UI に入力されたファイルの値を取得したら UI をリセットする方法を参考として置きますが、
      // これを行うよりも HTML フォーム内の "reset" 型の
      // input を使用します。
      //file_input.val(null);
      var file_url = $('#pub-file-url').val();
      if (selected_file) {
        addPublication(biblioid, title, year, selected_file);
      } else if (file_url) {
        addPublicationFromUrl(biblioid, title, year, file_url);
      } else {
        addPublication(biblioid, title, year);
      }

    });

    $('#delete-button').click(function(evt) {
      console.log("delete ...");
      var biblioid = $('#pub-biblioid-to-delete').val();
      var key = $('#key-to-delete').val();

      if (biblioid != '') {
        deletePublicationFromBib(biblioid);
      } else if (key != '') {
        // EcmaScript 6 エンジンでは use Number.isInteger を使用するとよい
        if (key == '' || isNaN(key))  {
          displayActionFailure("Invalid key");
          return;
        }
        key = Number(key);
        deletePublication(key);
      }
    });

    $('#clear-store-button').click(function(evt) {
      clearObjectStore();
    });

    var search_button = $('#search-list-button');
    search_button.click(function(evt) {
      displayPubList();
    });

  }

  openDb();
  addEventListeners();

})(); // Immediately-Invoked Function Expression (IIFE)</pre>

<p>{{LiveSampleLink('Full_IndexedDB_example', "オンラインのライブデモを試す")}}</p>

<div class="notecard note">
<p><strong>メモ</strong>: <code>window.indexedDB.open()</code> は非同期です。このメソッドは <code>success</code> イベントが発行されるよりもはるかに前に終了します。すなわち、ある関数 (例えば <code>openDb()</code>) が <code>open()</code> や <code>onsuccess</code> を呼び出すと、 <code>onsuccess</code> ハンドラーが実行されるよりも前に戻ります。この問題は、 <code>transaction()</code> や <code>get()</code> のような他のリクエストメソッドでも言えます。</p>
</div>

<h2 id="See_also">関連情報</h2>

<p>必要に応じて、より多くの情報を知るための記事集です。</p>

<h3 id="Reference">リファレンス</h3>

<ul>
 <li><a href="/ja/docs/Web/API/IndexedDB_API">IndexedDB API リファレンス</a></li>
 <li><a href="https://www.w3.org/TR/IndexedDB/">Indexed Database API Specification</a></li>
 <li>IndexedDB <a class="link-https" href="https://searchfox.org/mozilla-central/search?q=dom%2FindexedDB%2F.*%5C.idl&path=&case=false&regexp=true">インターフェイスファイル</a> (Firefox のソースコード内)</li>
</ul>

<h3 id="Tutorials_and_guides">チュートリアルとガイド</h3>

<ul>
 <li><a href="https://www.html5rocks.com/en/tutorials/indexeddb/uidatabinding/">Databinding UI Elements with IndexedDB</a></li>
 <li><a href="https://msdn.microsoft.com/en-us/scriptjunkie/gg679063.aspx">IndexedDB — The Store in Your Browser</a></li>
</ul>

<h3 id="Libraries">ライブラリー</h3>

<ul>
 <li><a href="https://localforage.github.io/localForage/">localForage</a>: クライアント側のデータストレージ向けに、シンプルな name:value 形式の構文を提供するポリフィルです。バックグラウンドで IndexedDB を使用しますが、IndexedDB に対応していないブラウザーでは WebSQL や localStorage にフォールバックします。</li>
 <li><a href="https://www.dexie.org/">dexie.js</a>: 優良でシンプルな構文により高速なコード開発を可能にする、IndexedDB のラッパーです。</li>
 <li><a href="https://github.com/erikolson186/zangodb">ZangoDB</a>: IndexedDB の MongoDB ライクなインターフェイスで、MongoDB でおなじみのフィルタリング、射影、ソート、アップデート、集計をサポートしています。</li>
 <li><a href="https://jsstore.net/">JsStore</a>: シンプルで高度な IndexedDB ラッパーで SQL ライクな文法があります。</li>
</ul>