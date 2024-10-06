---
title: "Shapefile形式をGeoJSON形式に変換してマップに表示する"
emoji: "🌏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GIS", "shapefile", "leaflet"]
published: true
---

## 始めに

マップにピンを立てるサンプルコードを作る際にダミーではなく何か良いデータがないかなぁと考えた時、以下のような施設の位置情報が使えるかなと思いました。
https://nlftp.mlit.go.jp/ksj/gml/datalist/KsjTmplt-P17.html

早速このデータをダウンロードしてみたのですが、`.shp`とか見慣れない拡張子があって中身を見ることができませんでした。。

色々調べてみるとどうやらGISデータと言われるものらしく、その中でもShapefileという形式であることを知りました。こちらはバイナリで保存されており、ソフトとか使わないと中身が見れないようです。

https://loris.co.jp/column/gis-data-formats.html

GISデータでもGeoJSONだとWebでも扱いやすいため、ShapefileからGeoJSONに変換するnpmパッケージがあるか調べたところ [shapefile](https://www.npmjs.com/package/shapefile) が使えそうだったのでそれを試してみました。またその結果をマップ上で確認できると嬉しいため、それも合わせて行ったのでそれらについて記事まとめました。

## ローカルでGeoJSON形式に変換する

### Node.jsでshapefileをGeoJSON形式に変換

shapefileでは`.shp`というファイルと`.dbf`というファイルが必要で、`shapefile`を使うと以下のように指定することで一緒に呼び出してGeoJSON形式に変換してくれます。ローカルでやる場合は`.dbf`の指定がなくても自動でこのファイルも探すらしいので指定は必須ではなさそうでした。

```ts:Node.jsでshapefileをGeoJSON形式に変換する
import shapefile from "shapefile";
import { fileURLToPath } from "node:url";
import path from "path";
import { promises as fsPromises } from "fs";

// CommonJSではなくESModuleで実行しているため、__dirnameの算出が必要
// @see https://zenn.dev/risu729/articles/dirname-in-esm
const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

const FILE_NAME = "P17-12_13_FireStation";

shapefile
  .read(
    path.resolve(__dirname, `./data/source/${FILE_NAME}.shp`),
    path.resolve(__dirname, `./data/source/${FILE_NAME}.dbf`),
    { encoding: "shift-jis" }
  )
  .then((geojson) => {
    fsPromises.writeFile(
      path.resolve(__dirname, `./data/output/${FILE_NAME}.geojson`),
      JSON.stringify(geojson, null, 2)
    );
  });
```

### 実行結果

こちらを実行した結果、GeoJSONデータは以下のようになりました。ちなみに`.dbf`ファイルが取り込まれない状態（ファイル名を変えるなどして同名ファイルを置かない状態）にして実行すると`properties`の部分が空になり、どうやらこういったプロパティ情報が`.dbf`ファイルにあるようです。

```json:shapefileをGeoJSONに変換したデータ
{
  "type": "FeatureCollection",
  "features": [
    {
      "type": "Feature",
      "properties": {
        "P17_001": "東京消防庁",
        "P17_002": "13101",
        "P17_003": 1,
        "P17_004": "千代田区大手町1-3-5"
      },
      "geometry": {
        "type": "Point",
        "coordinates": [139.761572, 35.688875]
      }
    },
    {
      "type": "Feature",
      "properties": {
        "P17_001": "稲城市消防本部",
        "P17_002": "13225",
        "P17_003": 1,
        "P17_004": "稲城市東長沼2111"
      },
      "geometry": {
        "type": "Point",
        "coordinates": [139.505013, 35.638043]
      }
    },
    ...
  ],
  "bbox": [139.092516, 33.114822, 139.909115, 35.804098]
}
```

geojsonデータは実はGitHub上だとプレビュー表示ができるようで、先ほどの出力結果をGitHubに上げましたのでマップ上でピンの位置を確認することができます。

![](/images/shapefile-parser/fire-station-preview.png)
[P17-12_13_FireStation.geojson](https://github.com/TakanoriOnuma/shapefile-parser/blob/main/data/output/P17-12_13_FireStation.geojson)

座標だけでなく領域情報もありましたので合わせて載せておきます。

![](/images/shapefile-parser/fire-station-jurisdiction-preview.png)
[P17-12_13_FireStationJurisdiction.geojson](https://github.com/TakanoriOnuma/shapefile-parser/blob/main/data/output/P17-12_13_FireStationJurisdiction.geojson)

### 参考記事

https://qiita.com/frogcat/items/b3235c06d64cee01fa47
https://qiita.com/frogcat/items/051572ccf92e084db378
https://zenn.dev/yuichiyazaki/articles/b70da4bb63ea02

## WebでGeoJSON形式に変換し、マップに表示する

GitHub上でGeoJSONファイルをプレビューできるのは素晴らしいなと思いつつ、これができるのであれば自分でも表示できるのでは？と思い調べたところ、[leaflet](https://leafletjs.com/) が使えそうでした。前セクションで使用した`shapefile`はWebでも使えるようなので、これらを組み合わせてshapefileをGeoJSONに変換してプレビュー表示し、それを見てGeoJSONデータをダウンロードするWebアプリを作りました。

### 作ったWebアプリ

先に作ったWebアプリを紹介しますと以下のような機能を持つものを作りました。

- input file要素でshapefileまたはGeoJSONのファイルを複数アップロードし、それをgeojsonに変換してマップ上に表示
- マップ上に表示するデータはチェックボックスで切り替えることが可能
- shapefileの場合はそれぞれGeoJSONとしてダウンロードが可能で、かつマップ上に表示しているデータを一つのGeoJSONデータとしてダウンロードすることも可能

スクショを貼ると以下のようなもので、アプリとリポジトリのURLもそれぞれ貼りますので興味がある方は是非見てください。

![](/images/shapefile-parser/shapefile-previewer.png)

https://takanorionuma.github.io/shapefile-parser/

https://github.com/TakanoriOnuma/shapefile-parser

次から各機能の実装について説明します。

### アップロードされたshapefileをGeoJSONに変換

shapefileをGeoJSONに変換するには`.shp`と`.dbf`それぞれが必要になります。ローカルでは暗黙的に同じファイル名があると自動で`.dbf`も読み込んでくれるようですがWebではそうはいきません。仮に`.shp`と`.dbf`の順で一括でアップロードした場合は次のようなコードで実装できます。

```tsx:.shpと.dbfをアップロードしてGeoJSONに変換する
import * as shapefile from "shapefile";

const loadBinaryFile = (file: File): Promise<ArrayBuffer> => {
  return new Promise((resolve) => {
    const reader = new FileReader();
    reader.onload = () => {
      const result = event.target?.result;
      if (result == null) {
        return;
      }
      resolve(result as ArrayBuffer);
    };
    reader.readAsArrayBuffer(file);
  });
};

export const GeoFileLoader = () => {
  return (
    <input
      value=""
      type="file"
      multiple
      onChange={async (event) => {
        const { files } = event.target;
        if (files == null) {
          return;
        }
        // .shp, .dbfの順番でファイルアップロードされたと仮定した時の処理
        const [shpFile, dbfFile] = files;

        // それぞれバイナリファイルとして読み込む
        const [shpData, dbfData] = await Promise.all([
          loadBinaryFile(shpFile),
          loadBinaryFile(dbfFile),
        ]);
        // バイナリデータをshapefileとして読み込んでGeoJSONを取得する
        const geojson = await shapefile.read(shpData, dbfData, {
          encoding: "shift-jis",
        });
      }}
    />
  );
};
```

今回作ったアプリでは`.shp`アップロード後に`.dbf`をアップロードしたら事前にアップロードしたファイルと組み合わせてshapefileとして扱ったり、`.dbf`ファイルがなくても`properties`がないだけでマップに表示することは可能なので`.shp`のみで登録することが可能だったり、変換処理が不要な`.geojson`や`.json`をアップロードすることができるようになっています。これらの実装は本筋から逸れるので詳細のコードを見たい方はこちらのURLからご参照ください。

https://github.com/TakanoriOnuma/shapefile-parser/blob/main/src/components/GeoFileLoader.tsx

### GeoJSONデータをマップに表示

まずはマップを表示する必要がありますが、`leaflet`を使うと以下のように書くと表示できます。GoogleMapのようにCtrlまたはCommandキーが入力中の時のみスクロールズームを有効にしたかったのですが、オプションがなかったので自前で設定しています。
ちなみにReact版の[react-leaflet](https://react-leaflet.js.org/)というライブラリもありましたが、細かいハンドリングをする場合にやりづらくなりそうな気がしたのでpureなものを使っております。

```tsx:leafletでマップを表示
import { FC, useMemo, useState, useEffect } from "react";
import L from "leaflet";

export type MapProps = {};

export const Map: FC<MapProps> = () => {
  const [map, setMap] = useState<L.Map | null>(null);

  const ref = useMemo(() => {
    let map: L.Map | null = null;
    const handleKeyDown = (event: KeyboardEvent) => {
      // CtrlキーまたはCommandキーが押されている場合はスクロールズームを有効にする
      if (event.ctrlKey || event.metaKey) {
        map?.scrollWheelZoom.enable();
      }
    };
    const handleKeyUp = () => {
      map?.scrollWheelZoom.disable();
    };
    return (element: HTMLElement | null) => {
      if (element == null) {
        map?.remove();
        setMap(null);
        document.removeEventListener("keydown", handleKeyDown);
        document.removeEventListener("keyup", handleKeyUp);
        return;
      }

      map = L.map(element, {
        scrollWheelZoom: false,
      });
      document.addEventListener("keydown", handleKeyDown);
      document.addEventListener("keyup", handleKeyUp);

      map.setView([35.681236, 139.767125], 15);
      const layer = L.tileLayer(
        "https://tile.openstreetmap.org/{z}/{x}/{y}.png",
        {
          // 著作権の表示
          attribution:
            '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
        }
      );
      map.addLayer(layer);

      setMap(map);
    };
  }, []);

  return (
    <div>
      <div
        ref={ref}
        style={{
          aspectRatio: "16 / 9",
        }}
      ></div>
    </div>
  );
};
```

ここからGeoJSONデータを読み込んで表示する場合は以下のようになります。`L.geoJSON`でGeoJSONデータを読み込んだレイヤーが出来上がるので非常に実装が楽でした。`L.geoJSON`にあるオプションはそれぞれ以下の設定を入れています。

- pointToLayer: マーカーのアイコン設定でデフォルトであれば本来必要ないが、Viteでビルドした際にアイコン画像がリンク切れしてしまったので明示的にアイコン画像をimportして使うようにした
- onEachFeature: 全てのFeatureデータに対してpropertiesがある場合はクリック時にポップアップ表示がされるように`layer.bindPopup`を設定

```diff tsx:GeoJSONデータをマップに表示
 import { FC, useMemo, useState, useEffect } from "react";
 import L from "leaflet";
+import markerIcon from "leaflet/dist/images/marker-icon.png";
+import markerIcon2x from "leaflet/dist/images/marker-icon-2x.png";
+import markerShadow from "leaflet/dist/images/marker-shadow.png";

+const isActive = (map: L.Map) => {
+  // 既にマップが削除されている場合はpanesが空になっているので、それで判断する
+  return Object.keys(map.getPanes()).length > 0;
+};

 export type MapProps = {
+  /** Geo JSONデータ */
+  geoJsonList: GeoJSON.GeoJSON[];
 };

 export const Map: FC<MapProps> = ({ geoJsonList }) => {
   const [map, setMap] = useState<L.Map | null>(null);

   // refの定義は省略

+  useEffect(() => {
+    if (map == null || !isActive(map) || geoJsonList.length <= 0) {
+      return;
+    }
+
+    const layer = L.geoJSON(geoJsonList, {
+      pointToLayer: (_, latlng) => {
+        return L.marker(latlng, {
+          // デフォルトアイコンをそのまま使うとビルド時に画像がリンク切れになってしまったのでimportして使用する
+          // @see https://github.com/Leaflet/Leaflet/blob/v1.9.4/src/layer/marker/Icon.Default.js#L22-L31
+          icon: L.icon({
+            iconUrl: markerIcon,
+            iconRetinaUrl: markerIcon2x,
+            shadowUrl: markerShadow,
+            iconSize: [25, 41],
+            iconAnchor: [12, 41],
+            popupAnchor: [1, -34],
+            tooltipAnchor: [16, -28],
+            shadowSize: [41, 41],
+          }),
+        });
+      },
+      onEachFeature: (feature, layer) => {
+        const entries = Object.entries(feature.properties);
+        if (entries.length > 0) {
+          layer.bindPopup(
+            entries.map(([key, value]) => `${key}: ${value}`).join("<br>")
+          );
+        }
+      },
+    });
+
+    map.addLayer(layer);
+    const bounds = layer.getBounds();
+    if (bounds.isValid()) {
+      map.fitBounds(bounds);
+    }
+
+    return () => {
+      map.removeLayer(layer);
+    };
+  }, [map, geoJsonList]);

   return (
     // returnするDOMは同じなので省略
   );
 };
```

これで最低限の機能はできましたが、GitHubのプレビューのようにクラスタリングもされていると良いなと思ったため、その設定も入れます。クラスタリングはパッケージが別で[`leaflet.markercluster`](https://github.com/Leaflet/Leaflet.markercluster)をimportすると`L.markerClusterGroup`が使用できるようになります。これはLayerでmapと同じようにaddLayerが使えるのでmapの代わりにこっちにaddLayerすることで自動でクラスタリングされます。

```diff tsx:クラスタリングの設定
 import { FC, useMemo, useState, useEffect } from "react";
 import L from "leaflet";
+import "leaflet.markercluster";

 // 一部省略

 export const Map: FC<MapProps> = ({ geoJsonList }) => {
   const [map, setMap] = useState<L.Map | null>(null);
+  const [markerClusterLayer, setMarkerClusterLayer] =
+    useState<L.MarkerClusterGroup | null>(null);

   const ref = useMemo(() => {
     let map: L.Map | null = null;
     // key入力ハンドラは同じなので省略
     return (element: HTMLElement | null) => {
       if (element == null) {
         map?.remove();
         setMap(null);
+        setMarkerClusterLayer(null);
         document.removeEventListener("keydown", handleKeyDown);
         document.removeEventListener("keyup", handleKeyUp);
         return;
       }

       map = L.map(element, {
         scrollWheelZoom: false,
       });
       // key入力イベントの設定やマップレイヤーの設定は同じなので省略

+      const markerClusterLayer = L.markerClusterGroup();
+      map.addLayer(markerClusterLayer);

       setMap(map);
+      setMarkerClusterLayer(markerClusterLayer);
     };
   }, []);

   useEffect(() => {
     if (
       map == null ||
       !isActive(map) ||
+      markerClusterLayer == null ||
       geoJsonList.length <= 0
     ) {
       return;
     }

     const layer = L.geoJSON(geoJsonList, {
       // optionの中身は一緒なので省略
     });
-    map.addLayer(layer);
+    markerClusterLayer.addLayer(layer);

-    const bounds = layer.getBounds();
+    const bounds = markerClusterLayer.getBounds();
     if (bounds.isValid()) {
       map.fitBounds(bounds);
     }

     return () => {
-      map.removeLayer(layer);
+      markerClusterLayer.removeLayer(layer);
     };
-   }, [map, geoJsonList];
+   }, [map, markerClusterLayer, geoJsonList]);

   return (
     // returnするDOMは同じなので省略
   );
 };
```

### マップ上のデータをGeoJSON形式で保存

`L.geoJSON`や`L.markerClusterGroup`で生成されたものは`.toGeoJSON`でGeoJSONデータを取得することができるので、以下のように書くことでダウンロードすることができます。

```diff tsx:markerClusterGroupのデータをGeoJSON形式で保存する
 // 省略

+/**
+ * GeoJsonファイルをローカルに保存する
+ * @param fileName - ファイル名
+ * @param json - GeoJSONデータ
+ */
+export const saveGeoJsonFile = (fileName: string, json: GeoJSON.GeoJSON) => {
+  const aElement = document.createElement("a");
+  const blob = new Blob([JSON.stringify(json, null, 2)], {
+    type: "application/json",
+  });
+  aElement.href = window.URL.createObjectURL(blob);
+  aElement.setAttribute("download", fileName);
+  aElement.click();
+  window.URL.revokeObjectURL(aElement.href);
+};

 // 省略

 export const Map: FC<MapProps> = ({ geoJsonList }) => {
   const [map, setMap] = useState<L.Map | null>(null);
   const [markerClusterLayer, setMarkerClusterLayer] =
     useState<L.MarkerClusterGroup | null>(null);

   // 省略

   return (
     <div>
       <div
         ref={ref}
         style={{
           aspectRatio: "16 / 9",
         }}
       ></div>
+      {markerClusterLayer && (
+        <button
+          style={{ marginTop: 5 }}
+          disabled={geoJsonList.length <= 0}
+          onClick={() => {
+            saveGeoJsonFile("output.geojson", markerClusterLayer.toGeoJSON());
+          }}
+        >
+          マップ上に表示しているデータをgeojsonとしてダウンロード
+        </button>
+      )}
     </div>
   );
 };
```

GeoJSON形式のデータを保存する機能はshapefileから変換したものもありますが、こちらは既にjsonデータになっておりそれを`saveGeoJsonFile`を呼び出すだけで良いので実装内容についての説明は割愛させていただきます。

### 参考記事

https://qiita.com/asahina820/items/7ea0ac3fc2fbbbe7512a
https://qiita.com/mitch0807/items/52698a561d4255578657

## 終わりに

以上がShapefile形式をGeoJSON形式に変換してマップにプレビュー表示し、変換データを保存する方法でした。Shapefileはバイナリであるためシュッと中身を見るのが難しいものですが`shapefile`と`leaflet`を使うことで簡単にプレビューできて、かつダウンロードもできるというかなり良さげなWebアプリができたんじゃないかなと思います。shapefileのデータを扱う際の参考になったり、単純に閲覧したい方の助けになれば幸いです。
