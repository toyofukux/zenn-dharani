---
title: "React Three FiberでOrbitControlsのカメラを操作できない問題への対処"
emoji: "🐛"
type: "tech"
topics: ["threejs", "React", "reactthreefiber", "r3f", "drei"]
published: true
---

ハマったのでメモ。
React Three Fiber を使って React で Three.js を使用している場合にうまくカメラを同期させないと、useEffect とかでカメラのパラメーターが操作できなくてハマりましたという話。

# 最初のコード

以下のソースだと、うまく OrbitControls がうまく作動しない。
`useFrame`とかで camera の ID を見ると、`Drei.PerspectiveCamera`と`OrbitControls.object`の ID と異なっていることがわかる。  
結果としてコンポーネントに渡している cameraState によるカメラ操作が適用されない。

```tsx
const CustomOrbitControls: React.FC<Props> = ({ cameraState }) => {
  const {
    gl: { domElement },
  } = useThree();

  const ref = useRef<OrbitControlsImpl>(null);
  const cameraRef = useRef<PerspectiveCamera>(null);

  useLayoutEffect(() => {
    // カメラの初期ポジション
    if (cameraRef && cameraRef.current) {
      cameraRef.current.position.set(50, 100, 300);
    }
  }, []);

  useEffect(() => {
    if (cameraRef && cameraRef.current) {
      // 親からもらったstateでカメラを操作したい
      cameraRef.current.fov = cameraState.fov;
    }
  }, [cameraState]);

  return (
    <>
      <Drei.PerspectiveCamera ref={cameraRef} {...cameraState} />
      <OrbitControls
        ref={ref}
        args={[cameraRef.current, domElement]}
        camera={cameraRef.current}
      />
    </>
  );
};
export default CustomOrbitControls;
```

# 修正後のコード

`Drei.PerspectiveCamera`に`makeDefault`をつけ、default のカメラを同期する。
drei コンポーネントは内部的に`useThree`を使っており、default のカメラを制御できないと操作できなくて死ぬ。

```tsx
const CustomOrbitControls: React.FC<Props> = ({ cameraState }) => {
  const {
    gl: { domElement },
  } = useThree();

  const ref = useRef<OrbitControlsImpl>(null);
  const cameraRef = useRef<PerspectiveCamera>(null);

  useLayoutEffect(() => {
    if (cameraRef && cameraRef.current) {
      cameraRef.current.position.set(50, 100, 300);
    }
  }, []);

  useEffect(() => {
    if (cameraRef && cameraRef.current) {
      cameraRef.current.fov = cameraState.fov;
    }
  }, [cameraState]);

  return (
    <>
      {/* makeDefaultをつける!! */}
      <Drei.PerspectiveCamera makeDefault ref={cameraRef} {...cameraState} />
      <OrbitControls
        ref={ref}
        args={[cameraRef.current, domElement]}
        camera={cameraRef.current}
      />
    </>
  );
};
export default CustomOrbitControls;
```

# まとめ

Three.js はもともと副作用のある処理が多く、React で利用する際は`useRef`を使いこなせないとかなり難易度が高い。

# 追記(2021-11-25)

drei のコミッターの方にモット短く書ける方法をアドバイスしてもらいました。

@[tweet](https://twitter.com/0xca0a/status/1463613566649569280)

```tsx
const CustomOrbitControls: React.FC<Props> = ({ cameraState }) => {
  const ref = useRef<OrbitControlsImpl>(null);
  const cameraRef = useRef<PerspectiveCamera>(null);

  useLayoutEffect(() => {
    if (cameraRef && cameraRef.current) {
      cameraRef.current.position.set(50, 100, 300);
    }
  }, []);

  useEffect(() => {
    if (cameraRef && cameraRef.current) {
      cameraRef.current.fov = cameraState.fov;
    }
  }, [cameraState]);

  return (
    <>
      {/* makeDefaultをつける!! */}
      <Drei.PerspectiveCamera makeDefault ref={cameraRef} {...cameraState} />
      <OrbitControls ref={ref} makeDefault />
    </>
  );
};
export default CustomOrbitControls;
```
