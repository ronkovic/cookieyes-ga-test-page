# 親ページとiframe間の同意情報連携設定 (postMessage API利用)

CookieYes等で管理されている親ページの同意情報を、iframe内のコンテンツ（GA/GTM）に連携させるための一般的な方法として、`postMessage` APIを利用する手順を以下に示します。

## 1. 基本的な仕組み

1.  **親ページ:**
    *   CookieYesなどによってユーザーの同意状態が変更・確定されたタイミングで、現在の同意情報を取得します。
    *   取得した同意情報を、`window.postMessage()` メソッドを使用してiframeに送信します。
2.  **iframeページ:**
    *   `window.addEventListener('message', callback)` を使用して、親ページからのメッセージを待ち受けます。
    *   受信したメッセージのオリジン（送信元）を検証し、信頼できる親ページからのものであることを確認します。
    *   受信した同意情報に基づき、iframe内のGA/GTMの同意状態 (`gtag('consent', 'update', ...)`など) を更新します。

## 2. 親ページ (`index.html`) での実装例

親ページのJavaScriptに以下の機能を追加します。

*   **同意情報をiframeに送信する関数:**
    *   CookieYesの同意状態が更新された際に呼び出されます。
    *   `iframeElement.contentWindow.postMessage({ type: 'consentUpdate', consent: consentObject }, targetOrigin);` のようにメッセージを送信します。
    *   `targetOrigin` はiframeのオリジン（例: `https://your-iframe-domain.com`）を指定し、セキュリティを確保します。
*   **同意更新の検知:**
    *   **重要:** CookieYesのAPIドキュメントを確認し、同意状態の変更を検知する正確な方法（カスタムイベント、コールバック関数など）を特定し、上記関数をトリガーする必要があります。
    *   代替案として、`dataLayer` にプッシュされるGTMの同意更新イベントを監視し、それをトリガーにすることも考えられます（CookieYesとGTM同意モードが連携している場合）。

```html
<!-- index.html の <script> タグ内 (概念例) -->
<script>
  // (既存のdataLayerやgtagのコード)

  function sendConsentToIframe(consentData) {
    const iframe = document.querySelector('iframe[src="ga_iframe_content.html"]');
    if (iframe && iframe.contentWindow) {
      // targetOriginはiframeの実際のオリジンを指定
      const targetOrigin = new URL(iframe.src).origin; 
      iframe.contentWindow.postMessage({ type: 'consentUpdate', consent: consentData }, targetOrigin);
      console.log('Parent: Sent consent to iframe:', consentData);
    }
  }

  // CookieYesの同意更新を検知するロジック (CookieYesの仕様に依存)
  // 例: window.addEventListener('cookieyes_consent_updated', (event) => {
  //   sendConsentToIframe(event.detail.consent); 
  // });

  // または、dataLayer経由で検知する例
  const originalPush = window.dataLayer.push;
  window.dataLayer.push = function(...args) {
    originalPush.apply(window.dataLayer, args);
    args.forEach(arg => {
      if (typeof arg === 'object' && arg && arg[0] === 'consent' && arg[1] === 'update') {
        sendConsentToIframe(arg[2]);
      }
    });
  };
</script>
```

## 3. iframeページ (`ga_iframe_content.html`) での実装例

iframeページのJavaScriptに以下の機能を追加します。

*   **親からのメッセージ受信リスナー:**
    *   `window.addEventListener('message', function(event) { ... });` を設定。
*   **オリジン検証:**
    *   `event.origin` をチェックし、期待される親ページのオリジンからのメッセージのみを処理します。
    *   `if (event.origin !== 'https://your-parent-domain.com') return;`
*   **同意状態の適用:**
    *   受信した同意情報 (`event.data.consent`) を使用して、iframe内の `gtag('consent', 'update', consentState);` を呼び出すか、GTMの `dataLayer` にプッシュしてGTM側で同意状態を更新します。

```html
<!-- ga_iframe_content.html の <script> タグ内 (概念例) -->
<script>
  // (既存のdataLayerやgtagのコード)

  window.addEventListener('message', function(event) {
    // expectedOriginは親ページの実際のオリジンを指定
    const expectedOrigin = 'https://ronkovic.github.io'; // 例
    
    // オリジンとメッセージタイプを検証
    if (event.origin !== expectedOrigin || !event.data || event.data.type !== 'consentUpdate') {
      // console.warn('Iframe: Message from unexpected origin or wrong type:', event.origin);
      return;
    }

    const consentState = event.data.consent;
    console.log('Iframe: Received consent from parent:', consentState);

    // iframe内のGA/GTMの同意状態を更新
    if (window.gtag) {
      window.gtag('consent', 'update', consentState);
      console.log('Iframe: gtag consent updated.');
    }
    // 必要に応じて、GTMのdataLayerにも情報を送る
    // window.dataLayer.push({'event': 'iframe_consent_received', 'consentDetails': consentState});
  });
</script>
```

## 4. 実装とテストの注意点

*   **CookieYesのAPI/イベント:** 親ページで同意更新を検知する方法は、CookieYesの公式ドキュメントで正確な情報を確認してください。
*   **オリジン指定:** `postMessage` の `targetOrigin` およびメッセージ受信時の `event.origin` の検証は、セキュリティ上非常に重要です。常に具体的なオリジンを指定し、`"*"` の使用は避けてください。
*   **同意オブジェクトの形式:** 親からiframeへ渡す同意情報のオブジェクト形式が、`gtag('consent', 'update', ...)` が期待する形式と一致していることを確認してください（例: `{'analytics_storage': 'granted'}`）。
*   **デバッグ:** ブラウザの開発者コンソールのログを活用して、メッセージの送受信や同意状態の更新が正しく行われているかを確認します。GTMのプレビューモードも併用します。
*   **タイミング:** iframeの読み込み完了前に親ページからメッセージが送信されると受信できない可能性があるため、iframeの準備ができてからメッセージを送信するような工夫（例: iframeからの準備完了メッセージを待つ）が必要になる場合もあります。

この`postMessage`を利用した連携は、多くのCMP（Consent Management Platform）とiframeコンテンツ間の同意同期で採用されている標準的なアプローチの一つです。
