## StickyHeaderやCarousel等のスクロール連動UIをSwiftUIで実装する際の状態管理の勘所

<p align="right">
<strong>酒井文也 (Fumiya Sakai) Twitter &amp; github: @fumiyasac</strong>
</p>

<hr>

## はじめに 

SwiftUIでStickyHeaderやCarousel、タブと連動したスクロール表現を実装する際に、見た目の再現自体はできたものの、ドラッグ中の値の扱い方や座標の反映タイミングが思ったより複雑で、気づいたら@Stateの数が増えすぎていたという経験をお持ちの方もいらっしゃるのではないでしょうか。

筆者自身も、UIKitやAndroidでスクロール連動UIを作ってきた経験があるにもかかわらず、SwiftUIで同じような表現を実装しようとした際に、DragGestureの一時値と確定値をどう分けるか、GeometryReaderから取得した座標をどのタイミングで表示へ反映するかといった判断で何度か手が止まった経験があります。

本稿では、過去に作成したSwiftUIの実装サンプルを題材に、スクロール連動UIを実装する際に繰り返し向き合うことになる状態管理の判断ポイントをご紹介できればと思います。最新APIの紹介ではなく、実装中に __「ここはどうするか？」__ と迷いやすい部分を取り上げて、次のUI実装で判断材料にできるポイントをお伝えします。

## ⭐️Case 1: DragGestureの一時値と確定値を分けて持つ

CarouselやTinder風カードスワイプのようなDragGestureを利用した表現では、ドラッグ中に変化し続ける値と、指を離した後に確定する値を1つの@Stateにまとめて持つと、操作を途中でキャンセルした際に元の位置へ戻れなくなったり、スナップ先の計算がずれたりします。

Carouselの実装では次の2つを用途別に分けて持つことがポイントになります。`snappedOffset`は指を離した時点でのCarousel位置を記録する確定値、`draggingOffset`はドラッグ中の変化量をリアルタイムに反映する一時値です。

__【🌷Carousel実装で使用する@State変数の宣言例】__

```swift
// ※ Carousel表示要素のModifierへ適用する値の計算に使用する

// 【確定値】指を離した瞬間のCarousel位置を記録する
// 👉 .onEndedで更新され、次のドラッグ開始時の基準点になる
@State private var snappedOffset: CGFloat = 0

// 【一時値】ドラッグ中の変化量をリアルタイムに反映する
// 👉 .onChangedで常に更新され、Modifierの値計算に直接使用する
@State private var draggingOffset: CGFloat = 0
```

ドラッグ中（`.onChanged`）はdraggingOffsetのみを更新し、指を離した時（`.onEnded`）に丸めた値を`snappedOffset`へ反映します。

__【🌷DragGestureでの一時値・確定値の更新タイミングの処理例】__

```swift
.gesture(
    DragGesture(minimumDistance: 20)
        .onChanged { value in
            // ドラッグ中は一時値のみ更新する snappedOffset（前回確定した位置）を起点に現在の移動量を加算してリアルタイムに反映する
            // 👉 250で割ることでドラッグ感度を調整している
            draggingOffset = snappedOffset + value.translation.width / 250
        }
        .onEnded { value in
            // Step1: 指を離した時点の移動量を元に最終的な位置を算出する
            draggingOffset = snappedOffset + value.translation.width / 250

            // Step2: 要素数で割った余りを使って最も近い要素の位置へスナップさせる
            // 👉 remainder(dividingBy:)で-0.5〜0.5の範囲に丸める
            draggingOffset = round(draggingOffset)
                .remainder(
                    dividingBy: Double(itemCount)
                )

            // Step3: 確定値を更新して次のドラッグの起点にする
            // 👉 ここで初めてsnappedOffsetを更新する
            snappedOffset = draggingOffset
        }
)
```

Tinder風カードスワイプでも同じ考え方が登場します。こちらでは`swipeOffset`（一時値）と`swipeStatus`（確定値としての状態）を分けて持ち、X軸方向の変化量がしきい値を超えた時に初めて`swipeStatus`を更新します。

しきい値の判定にはデバイス幅に対する変化量の割合を使うことで、端末サイズに依存しない実装になります。

__【🌷Tinder風カードスワイプのしきい値判定と状態更新の処理例】__

```swift
// 【確定値】カードの操作状態を管理する（.none / .addToCart / .notSelected）
@State private var swipeStatus: SwipeStatus = .none

// 【一時値】ドラッグ中のX軸・Y軸の移動量を保持する
@State private var swipeOffset: CGSize = .zero

// しきい値：デバイス幅に対する変化量の割合で判定する
// 👉 絶対値ではなく割合を使うことで端末サイズ差を吸収する
private let thresholdActionPercentage: CGFloat = 0.45

.gesture(
    DragGesture()
        .onChanged { value in
            // ドラッグ中は一時値のみを更新する
            swipeOffset = value.translation
        }
        .onEnded { value in
            // X軸方向の変化量の割合がしきい値を超えた時のみ確定値（swipeStatus）を更新する
            let xPercentage = abs(value.translation.width)
                / UIScreen.main.bounds.width
            if xPercentage > thresholdActionPercentage {
                swipeStatus = value.translation.width > 0
                    ? .addToCart
                    : .notSelected
            } else {
                // しきい値未満はキャンセル扱いで一時値をリセットする
                swipeOffset = .zero
                swipeStatus = .none
            }
        }
)
```

また、`draggingOffset`はジェスチャーの管理だけでなく、各Carousel要素のscaleEffect・opacity・zIndexといったModifierの値を計算する際にも直接使用します。各要素と中央との距離を`draggingOffset`から算出し、その距離に応じた値をModifierへ適用することで奥行きのある表現になります。

__【🌷draggingOffsetを利用したCarousel要素のModifier値算出例】__

```swift
// 【Modifier値の算出】
// scaleEffect・opacityなどの値をdraggingOffsetからの距離で決める
// 👉 中央の要素（distance=0）が最大値1.0となり離れるほど小さくなる
private func getModifierValue(
    itemId: Int,
    ratio: CGFloat
) -> CGFloat {
    let distance = calculateDistance(itemId: itemId)
    return 1.0 - abs(distance) * ratio
}

// 【各要素のdraggingOffsetからの距離を算出する】
private func calculateDistance(itemId: Int) -> CGFloat {
    let offsetByItemId = draggingOffset - CGFloat(itemId)
    // 👉 remainder(dividingBy:)で要素数の範囲内に収める
    return offsetByItemId
        .remainder(dividingBy: CGFloat(itemCount))
}

// 👉 各ratioの値を変えることで奥行きや重なりの強さを調整できる
// （例: scaleEffect ratio=0.2 / opacity ratio=0.3 / zIndex ratio=1.0）
```

変数の名前や型は違っても __ドラッグ中は一時値だけを動かし、確定時に初めて状態へ反映する__ という設計の考え方は2つの表現に共通しています。

## ⭐️Case 2: GeometryReaderから取得した座標を@Stateへ反映するタイミング

StickyHeaderのようにScrollViewのオフセット値に応じてアニメーションさせる表現では、GeometryReaderで取得したフレームの値を`body`内で直接@Stateへ書き込もうとすると、「Modifying state during view update, this will cause undefined behavior」という警告が出て意図した動きになりません。

これはViewの描画サイクル中に@Stateを変更しようとしているために起きます。PreferenceKeyを経由して値を伝播させることで、描画サイクルの外で@Stateを更新できます。

__【🌷PreferenceKeyの定義とView Extensionを利用したオフセット値取得の処理例】__

```swift
// 【PreferenceKeyの定義】
// ScrollViewのオフセット値（minY）を子Viewから親Viewへ伝播させるためのKey
// 👉 reduce内の処理は空でよい（最後に設定された値をそのまま使うため）
fileprivate struct OffsetPreferenceKey: PreferenceKey {
    static var defaultValue: CGFloat = .zero
    static func reduce(
        value: inout CGFloat,
        nextValue: () -> CGFloat
    ) {}
}

// 【View Extension】
// 配置されたView要素のScrollView内での位置（minY）を取得して返すModifier
// 👉 overlayで透明なGeometryReaderを重ねることで、本来のViewのレイアウトへ影響を与えずに座標を取得している
extension View {
    func getScrollOffset(
        completion: @escaping (CGFloat) -> ()
    ) -> some View {
        self.overlay {
            GeometryReader { proxy in
                // .named("SCROLL")を指定することで
                // ScrollView全体を基準にした相対座標を取得する
                // 👉 .globalを使うと画面全体基準になり意図した値が取れない
                let minY = proxy.frame(
                    in: .named("SCROLL")
                ).minY
                Color.clear
                    // 取得したminYをPreferenceKeyへセットする
                    // 👉 この時点ではまだ@Stateは更新されない
                    .preference(
                        key: OffsetPreferenceKey.self,
                        value: minY
                    )
                    // 描画サイクルの外でPreferenceKeyの変化を受け取る
                    // 👉 ここで初めて@Stateを安全に更新できる
                    .onPreferenceChange(
                        OffsetPreferenceKey.self,
                        perform: completion
                    )
            }
        }
    }
}
```

`onPreferenceChange`のクロージャはViewの描画サイクルの外で呼ばれるため、ここで@Stateを更新しても警告は出ません。coordinateSpaceに`.named("SCROLL")`を指定している点も重要で、`.global`を使うと画面全体を基準にした座標になり、ScrollViewを基準にした相対オフセット値が取れなくなります。

実際にScrollViewへ組み込む際は、`.coordinateSpace(name: "SCROLL")`をScrollViewへ設定し、内部のコンテンツ要素に対して`getScrollOffset`を適用します。

__【🌷ScrollViewへのcoordinateSpace設定とPreferenceKey接続の実装例】__

```swift
// 【body内でのScrollView全体構造】
// 👉 @State var minY: CGFloat = 0 を別途宣言しておく
ScrollView(.vertical, showsIndicators: false) {
    VStack(spacing: 0) {
        // コンテンツ先頭に高さ0の透明Viewを置いてオフセットを取得する
        // 👉 getScrollOffsetのclosureで受け取った値を@Stateへ反映する
        Color.clear
            .frame(height: 0)
            .getScrollOffset { value in
                minY = value
            }
        // その他のコンテンツ要素を配置する
        ArticleView()
        // ...
    }
}
// 👉 .named("SCROLL")とgetScrollOffset内の指定を必ず揃える
.coordinateSpace(name: "SCROLL")
```

取得した`minY`を元にヘッダーの透過度や位置を計算する例は次のようになります。

__【🌷オフセット値からヘッダーの透過度・位置を算出する処理例】__

```swift
// 【透過度の算出】
// minYが負の値（上方向へスクロール済み）になるほどheaderOpacityが1.0に近づく
// 👉 headerHeight * 0.8の位置を100%の基準にすることでヘッダーが完全に隠れる前に透過アニメーションを完了させる
private var headerOpacity: CGFloat {
    let progress = -minY / (headerHeight * 0.8)
    // min/maxで0.0〜1.0の範囲に収める（範囲外になるとアニメーションが崩れる）
    return min(max(progress, 0), 1)
}

// 【オフセットの算出】
// 上方向へスクロールした分だけヘッダーを追従させて固定表示にする
// 👉 minYが正の値（下方向へスクロール）の場合は0を返して動かさない
private var headerOffset: CGFloat {
    minY < 0 ? -minY : 0
}
```

GeometryReaderで値を取得する場所と@Stateへ書き込む場所をPreferenceKeyで分離するこのパターンは、StickyHeader以外のスクロール連動アニメーションにも応用できます。 __直接書けないから経由させる__ という構造を一度理解しておくと、同じ警告に遭遇した際に対処の方針がすぐに立てられるようになります。

## ⭐️Case 3: StickyHeaderとタブ同期に共通するオフセット値の伝播パターン

StickyHeaderとタブ同期は見た目も用途も異なりますが、ScrollViewから取得したオフセット値を、別のView要素の表示状態へ伝播させるという共通の構造を持っています。

__StickyHeaderの場合__ は、Y軸方向のオフセット値がしきい値を下回った時点でヘッダーの表示形態を切り替えます。Case 2で紹介したPreferenceKeyのパターンを使ってオフセット値を取得し、その値を元に各Modifierへ適用する値を算出します。オフセット値そのものを@Stateとして持つのではなく、`headerOpacity`や`headerOffset`のように「表示に直結する値」へ変換した上でModifierへ渡すことで、Viewの再描画範囲を必要な箇所に絞ることができます。

__タブ同期の場合__ は、X軸方向のオフセット値から現在のタブインデックスを算出して@Stateへ反映します。スクロールが止まった時点で選択状態を確定させる点は、Case 1の確定タイミングと同じ考え方です。コンテンツをスワイプした時にタブの選択状態が連動して変わる表現でも、中間的なオフセット値を@Stateへ持ち込まず、インデックスという最小単位の値のみを管理することがポイントです。

StickyHeaderの構造では、Case 2で算出した`headerOpacity`と`headerOffset`をHeaderViewへ適用する部分が実際の接続点になります。

__【🌷 StickyHeader表現におけるHeaderViewへのModifier適用例】__

```swift
// 【HeaderView全体の構成例】
// 👉 var minY: CGFloat はCase 2のPreferenceKeyで更新される値
ZStack(alignment: .top) {
    ScrollView(.vertical, showsIndicators: false) {
        // ...コンテンツ要素...
    }
    .coordinateSpace(name: "SCROLL")
    // 👉 headerOpacityとheaderOffsetはCase 2で算出したComputed Property
    VStack {
        HeaderView()
            .opacity(headerOpacity)
            // 👉 これによりコンテンツに追従しながら上端に固定される
            .offset(y: headerOffset)
        Spacer()
    }
    .zIndex(1)
}
```

__【🌷タブインデックスの算出とスクロール停止時の選択状態確定の処理例】__

```swift
// 【タブインデックスの算出】
// X軸のオフセット値からどのタブが選択されているかを逆算する
// 👉 オフセットは左方向へのスクロールで負の値になるため符号を反転させてからtabWidthで割ることでインデックスを求める
private func selectedIndex(from offsetX: CGFloat) -> Int {
    let index = Int(round(-offsetX / tabWidth))
    // タブ数の範囲外にならないようmin/maxで制限する
    return min(max(index, 0), tabCount - 1)
}

// 【スクロール停止時にタブ選択状態を確定する】
// 👉 .interactingの間（指で操作中）はindexを確定させない → Case 1のDragGestureと同じ「確定タイミング」の考え方
// 👉 .idle（スクロールが完全に止まった時点）で初めて@Stateを更新する
.onScrollPhaseChange { oldPhase, newPhase in
    if newPhase == .idle {
        selectedTab = selectedIndex(
            from: scrollOffset
        )
    }
}
```

2つの表現に共通しているのは __ScrollViewのオフセット値を中間状態として持たず、表示に必要な値へ直接変換して@Stateへ渡す__ という考え方です。中間状態を増やすほど、どの値がどこから来ているかが追いにくくなります。

## まとめ

本稿で取り上げた3つのケースを振り返ると、それぞれに異なる形で「@Stateへ書くタイミングと粒度の判断」が求められていたことが分かります。3つのケースに共通して現れたのは、 __①一時値と確定値を分ける__ / __② PreferenceKeyで取得と更新を分離する__ / __③生の値を変換してから渡す__ という判断です。

3つのケースは __操作中か確定後か？__ / __描画サイクル中か外か？__ / __生の値か変換後の値か？__ という異なる問いに向き合っていますが、根本にあるのは「今この値を@Stateへ書いてよいタイミングか」という共通の問いです。UIKitでは「どこに書くか」が判断の中心でしたが、SwiftUIでは「いつ・何を・どの粒度で持つか」が実装の見通しと動作の安定性に直結します。なぜそう書くのかを整理しておくことで、見た目が異なる表現でも同じ判断軸で臨めるようになると感じています。本稿が、次のUI実装で手が止まる時間を少しでも減らすきっかけになれば幸いです。

- 『Case 1: DragGestureの一時値と確定値を分けて持つ』の参考リポジトリ
  - https://github.com/fumiyasac/CharacteristicStyleSwiftUIExample / https://github.com/fumiyasac/TinderCartExampleSwiftUI
- 『Case 2: GeometryReaderから取得した座標を@Stateへ反映するタイミング』の参考リポジトリ
  -  https://github.com/fumiyasac/LikeCoodinatorLayoutExample / https://github.com/fumiyasac/ScrollAnimationShowcase
- 『Case 3: StickyHeaderとタブ同期に共通するオフセット値の伝播パターン』の参考リポジトリ
  - https://github.com/fumiyasac/LikeCoodinatorLayoutExample / https://github.com/fumiyasac/ScrollableTabActionExample
