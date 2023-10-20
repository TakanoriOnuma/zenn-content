---
title: "キャンセル可能なAPIリクエストパッケージを作る"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["promise", "api"]
published: true
published_at: 2019-01-07
---

:::message
これは2019年01月07日にQiitaで作成した記事を移行したものになります。
:::

# 始めに

皆さんは通信中にキャンセルする機能を用意していますでしょうか。通信のキャンセルは普段必要ないと思いますが、ある状況下で困ってしまうケースが起きてしまうのではないのかなと思います。

- レスポンスが延々と返ってこない時があって通信帯域を逼迫させる
- 通信中にブラウザバックされたため通信をキャンセルしたい

レスポンス返ってこない問題はtimeoutを設定しておけば問題ないですが、通信中のブラウザバックはキャンセルしておかないと厳しいかなと思っています。

ただキャンセルを実装しようとすると変数が増えたり結構ややこしくなってしまいます（axiosもキャンセルトークンをセットしないといけないですし）。個人的にはPromiseにcancelメソッドがついていたらなぁと思っていますが、デフォルトではサポートしないようです。

https://blog.jxck.io/entries/2017-07-19/aborting-fetch.html

しかしcancel可能なPromiseがnpmパッケージとして作られているので、これを使ってcancelしたいと思います。

https://github.com/sindresorhus/p-cancelable

せっかくなのでクラスを作って、通信中のものを全て削除できるようにして、それをパッケージ化にするところまで実装しました。

# p-cancelableを返して通信をキャンセルさせる

axiosをラップして、promiseではなく、pCancelableを返します。pCancelableはpromiseを継承しているので、基本的にはPromiseと同じです。これにキャンセルメソッドがついただけです。
パラメータがかなり多くなってしまいましたが、汎用性のため仕方ないです。callbacksはフックするために用意しているだけなので、不要であれば使わなくて大丈夫です。

```js:request.js
import axios from 'axios';
import urlJoin from 'url-join';
import PCancelable from 'p-cancelable';

// APIメソッド
export const GET = 'get';
export const POST = 'post';
export const PUT = 'put';
export const DELETE = 'delete';

/**
 * API通信をする
 * @param {string} apiRoot - APIルート
 * @param {Object} options - APIオプション
 * @param {string} options.method - 通信メソッド名
 * @param {string} options.endpoint - 通信先
 * @param {Object?} options.query - 通信につけるクエリ
 * @param {Object?} options.header - 通信につけるヘッダー
 * @param {number?} options.timeout - 通信のタイムアウト
 * @param {Object} callbacks - コールバック関数群
 * @param {function?} callbacks.onRequestStart - リクエスト開始時のコールバック
 * @param {function?} callbacks.onSuccess - 成功時のコールバック
 * @param {function?} callbacks.onFailure - 失敗時のコールバック
 * @param {function?} callbacks.onCancel - キャンセル時のコールバック
 * @param {function?} callbacks.onRequestEnd - リクエスト終了時のコールバック
 * @returns {PCancelable} - キャンセル可能なPromise
 */
export function request(apiRoot, options, callbacks = {}) {
  const {
    method,
    endpoint,
    query = {},
    timeout = 15000
  } = options;
  const headers = {
    ...options.headers
  };

  const url = urlJoin(apiRoot, endpoint);

  // axiosでキャンセルするためにsourceを作る
  const source = axios.CancelToken.source();

  return new PCancelable((resolve, reject, onCancel) => {
    // リクエスト開始コールバックを呼ぶ
    callbacks.onRequestStart && callbacks.onRequestStart({ method, url });

    // requestのパラメータを生成する
    const requestOptions = {
      method,
      url,
      headers,
      timeout,
      cancelToken: source.token
    };
    requestOptions[method === GET ? 'params' : 'data'] = query;

    // リクエストの生成
    axios(requestOptions)
      .then((res) => {
        // リクエスト成功コールバックを呼ぶ
        callbacks.onSuccess && callbacks.onSuccess({ method, url }, res);
        resolve(res);
      })
      .catch((err) => {
        // キャンセルされた時のエラーは何もしない（onCancel側で処理を書く)
        if (axios.isCancel(err)) {
          return;
        }
        // リクエスト失敗コールバックを呼ぶ
        callbacks.onFailure && callbacks.onFailure({ method, url }, err);
        reject({
          isCancel: false,
          err
        });
      })
      .finally(() => {
        // リクエスト終了コールバックを呼ぶ
        callbacks.onRequestEnd && callbacks.onRequestEnd({ method, url });
      });

    // キャンセルを実行した時
    onCancel(() => {
      // キャンセルコールバックを呼ぶ
      callbacks.onCancel && callbacks.onCancel({ method, url });
      // 通信をキャンセルする
      source.cancel();
      reject({
        isCancel: true
      });
    });
  });
}
```

これで以下のようにしたら通信中にキャンセルができるようになります。簡単ですね！

```js:requestのサンプルコード
import { request, GET } from './request.js';

const pCancelable = request('http://localhost:8080', {
  method: GET,
  endpoint: '/data'
});

pCancelable
  .then((response) => {
    console.log(response);
  })
  .catch(({ isCancel, err }) => {
    // キャンセルしたかのフラグチェック
    if (isCancel) {
      console.log('canceled.');
      return;
    }
    console.error(err);
  });

// 1秒後にキャンセルする
window.setTimeout(() => {
  pCancelable.cancel();
}, 1000);
```

ただ1個だけ注意しなければいけないのが、requestを実行したら一回promiseを受け取らないといけないです。以下のように`.then().catch()`で繋げたものを受け取ってはいけません。意外と気づかないと思いますが、実は`.then()`や`.catch()`を実行するたびに新しいPromiseが作られています。新しく作られたPromiseはcancel可能なPromiseではなく、ただのPromiseなのでcancelメソッドが使えなくなります。

```js:失敗例
// pCancelableではなく普通のpromiseを受け取っている
const pCancelable = request('http://localhost:8080', {
  method: GET,
  endpoint: '/data'
})
  .then((response) => {
    console.log(response);
  })
  .catch(({ isCancel, err }) => {
    // キャンセルしたかのフラグチェック
    if (isCancel) {
      console.log('canceled.');
      return;
    }
    console.error(err);
  });

// cancelメソッドがないと怒られる
pCancelable.cancel();
```

これでキャンセルできるようになりました。ただキャンセルをするために全てのpCancelableを管理するのは大変のなので、それを管理するクラスを用意します。

# CancelableAPIクラスを作る

先ほど作ったrequestモジュールを使ってキャンセル可能なAPIクラスを作成します。このクラスでpCancelableリストを保持して、`cancelAll`を実行した時に通信中のpCancelableを全てキャンセルするようにします。

```js:CancelableAPI.js
import { request, GET, POST, PUT, DELETE } from './request';

// APIインスタンスリスト
const APIs = [];

/**
 * API通信のベースとなるクラス
 */
class CancelableAPI {
  // HTTPメソッド
  static GET = GET;
  static POST = POST;
  static PUT = PUT;
  static DELETE = DELETE;

  /**
   * コンストラクタ
   * @param {string} apiRoot - APIルート
   */
  constructor(apiRoot = '') {
    // APIルート
    this.apiRoot = apiRoot;
    // 通信中のcancelable promiseリスト
    this.pCancelableList = [];
    // APIインスタンスリストに登録する
    APIs.push(this);
  }

  /**
   * APIルートの設定
   * @param {string} apiRoot - APIルート
   */
  setAPIRoot(apiRoot) {
    this.apiRoot = apiRoot;
  }

  /**
   * API通信をする
   * @param {Object} requestOptions - リクエストオプション
   * @param {string} requestOptions.method - 通信メソッド名
   * @param {string} requestOptions.endpoint - 通信先
   * @param {Object?} requestOptions.query - 通信につけるクエリ
   * @param {Object?} requestOptions.header - 通信につけるヘッダー
   * @param {number?} requestOptions.timeout - 通信のタイムアウト
   * @param {Object} callbacks - コールバック関数群
   * @param {function?} callbacks.onRequestStart - リクエスト開始時のコールバック
   * @param {function?} callbacks.onSuccess - 成功時のコールバック
   * @param {function?} callbacks.onFailure - 失敗時のコールバック
   * @param {function?} callbacks.onCancel - キャンセル時のコールバック
   * @param {function?} callbacks.onRequestEnd - リクエスト終了時のコールバック
   * @returns {PCancelable} - キャンセル可能なPromise
   */
  request(requestOptions, callbacks = {}) {
    // pCancelableリストに登録する
    const pCancelable = request(this.apiRoot, requestOptions, callbacks);
    this.pCancelableList.push(pCancelable);

    pCancelable
      // catchしないとエラーメッセージが出てくるので受け取っておく
      .catch(() => {})
      // Promiseが終了した時にリストから外す
      .finally(() => {
        this.pCancelableList = this.pCancelableList.filter((promise) => promise !== pCancelable);
      });

    return pCancelable;
  }

  /**
   * 一つのインスタンスで実行された通信中のものを全てキャンセルする
   */
  cancelAll() {
    this.pCancelableList.forEach((pCancelable) => {
      pCancelable.cancel();
    });

    // cancelableのリストはrequestメソッド側で外れるが、先に外してしまう
    this.pCancelableList = [];
  }

  /**
   * 静的キャンセルメソッドで、全ての通信をキャンセルする
   */
  static cancelAll() {
    APIs.forEach((API) => {
      API.cancelAll();
    });
  }
}

export default CancelableAPI;
```

このクラスを継承してプロジェクトごとに拡張することで容易にキャンセルできるようになります。

```js:CancelableAPIを使った例
import CancelableAPI from 'CancelableAPI.js';

class BaseAPI extends CancelableAPI {
  /**
   * リクエストのテスト
   * @returns {PCancelable}
   */
  fetch() {
    return this.request({
      method: CancelableAPI.GET,
      endpoint: '/data'
    });
  }
}

// APIインスタンスを作る
const API = new BaseAPI('http://localhost:8080');

// リクエストを10回送る
for (let i = 0; i < 10; i++) {
  API.fetch()
    .then((response) => {
      console.log(response);
    })
    .catch(({ isCancel, err }) => {
      if (err) {
        console.error(err);
      }
    });
}

// 1秒後にまとめてキャンセルする
window.setTimeout(() => {
  API.cancelAll();
}, 1000);
```

複数のAPIで通信していて、全てのAPIをキャンセルしたい場合は`CancelableAPI.cancelAll()`を実行します。

```js:複数のAPIインスタンスをまとめてキャンセルする
// APIインスタンスを作る
const API1 = new BaseAPI('http://localhost:8080');
const API2 = new BaseAPI('http://localhost:10000');

API1.fetch();
API2.fetch();

// API1, API2それぞれで通信中のものをまとめてキャンセルする
CancelabelAPI.cancelAll();
```

pCancelableを返しているので当然個別でキャンセルすることもできます。

```js:個別でキャンセルする
const pCancelable = API.fetch();

pCancelabel.cancel();
```

# イベントをフックする

僕はよく通信開始や通信終了をstoreに保存して、通信中は上に透明なレイヤーを置いてタッチ操作を出来ないようにしています。そうしたフック処理を書けるようにcallbackを提供しています。この記事の最後にサンプルリポジトリを置いていますので、詳細はそちらの方で確認してください。

```js:イベントをフックしてstoreにcommitする
// CancelableAPIを使用する
import CancelableAPI from 'cancelable-api';

// storeにcommitする
import store from './store/';
import * as mutationTypes from './store/api/mutationTypes';

class API extends CancelableAPI {
  /**
   * API通信をする
   * @param {Object} requestOptions - リクエストオプション
   * @param {string} requestOptions.method - 通信メソッド名
   * @param {string} requestOptions.endpoint - 通信先
   * @param {Object?} requestOptions.query - 通信につけるクエリ
   * @param {Object?} requestOptions.header - 通信につけるヘッダー
   * @param {number?} requestOptions.timeout - 通信のタイムアウト
   * @returns {PCancelable} - キャンセル可能なPromise
   */
  request(requestOptions) {
    return super.request(requestOptions, {
      // 各イベントをフックする
      onRequestStart: ({ method, url }) => { store.commit(mutationTypes.REQUESTING, { method, url }); },
      onSuccess: ({ method, url }) => { store.commit(mutationTypes.SUCCESS, { method, url }); },
      onFailure: ({ method, url }) => { store.commit(mutationTypes.FAILURE, { method, url }); },
      onCancel: ({ method, url }) => { store.commit(mutationTypes.CANCEL, { method, url }); },
      // onRequestEnd: () => { console.log('request end'); }
    });
  }

  /**
   * リクエストのテスト
   * @returns {PCancelable}
   */
  fetch() {
    return this.request({
      method: CancelableAPI.GET,
      endpoint: '/exec'
    });
  }
}

export default API;
```

## ちなみに

APIからstoreにcommitする方法を書いていますが、逆にstoreへのactionを通じてAPIリクエストを送る場合はaction内で実行すればいいと思います。ただactionから返したものは一度Promiseでラップされた気がして、cancelメソッドが消えてしまった気がします・・・。
ただ全ての通信をキャンセルするだけならこれでもいいと思います。

```js:actionで通信を書く場合
export default {
  actions: {
    fetch({ state, commit }) {
      const pCancelable = API.fetch();
      pCancelable
        .then((response) => {
          commit('save', response);
        });
      // こうやっても結局promiseでラップされてしまった気がする
      return pCancelable;
    },
    // 全ての通信をキャンセルすることは出来る
    cancelAll() {
      API.cancelAll();
    }
  }
}
```

# パッケージ化する

折角なのでこのCancelabelAPIをパッケージ化しました。以下でインストールできます。

```
$ yarn add TakanoriOnuma/cancelable-api
```

リポジトリはこちらに置いています。
https://github.com/TakanoriOnuma/cancelable-api

パッケージ化の方法は以下を参考にしてください。

https://zenn.dev/numa_san/articles/3de3a8996cd6eb
https://zenn.dev/numa_san/articles/27da6899ade1a7

# サンプルコード

CancelableAPIを使ったサンプルコードは以下のリポジトリに置きました。興味がある方はぜひ見てください。

https://github.com/TakanoriOnuma/use-cancelable-api

# 終わりに

通信のキャンセルって真面目にやると相当めんどくさいと思います。ただ後回しにして、いざ問題が起きて対応しなければいけなくなると相当辛い思いをすると思います。
今回は普段使っているときはいつも通りに使えて、キャンセルしたくなったら容易にキャンセルできるような設計を心がけました。ひとえにキャンセルといっても全てキャンセルしたり、あるグループだけキャンセルしたり、個別でキャンセルしたりと、パターンが非常に多く使用方法も複雑になってしまいました。ただ基本的には使い勝手はいいんじゃないかなと思ってはいます。
パッケージは一応公開していますが、いつ消えるか分からないので実運用では使わないようにお願いします（念の為）。これをforkしてブラッシュアップする分には全然問題ないです。
皆さんもこれを参考に通信をキャンセルする仕組みを考えていただけたら幸いです。
