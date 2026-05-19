# ChromePublic APK 2026-04-27 指纹一致性真机回归报告

## Release / APK

| 项 | 值 |
|---|---|
| Release Tag | `apk-20260427-2488269d4d52` |
| APK 文件 | `ChromePublic-creepjs-battery-media-fixed.apk` |
| 本地路径 | `.tmp/apk/ChromePublic-creepjs-battery-media-fixed.apk` |
| Package | `org.bromite.bromite` |
| Version | `149.0.7779.0` |
| VersionCode | `777900004` |
| Size | `472472538` bytes |
| SHA-256 | `2488269d4d520a162498d91b1a38e1afa6630dab14f7ef0a8b05b13ac26f64d0` |
| 真机 | `5c8093e4 / Xiaomi 2509FPN0BC` |
| 手机浏览器出口 IP | `84.245.9.218` |

## 本次修复点

- `generate_spoof_profile.py` 生成阶段归一化 Android Chromium WebGL numeric params，避免数据库抽到 `16383/4096/1,4095.9375/1,1024` 这类和当前 APK 实测不一致的值。
- `battery.level` 改为 JS Battery API 形态的 `0-1` 小数，例如 `0.57`，不再写 `57` 这种百分数。
- `--public-ip` 使用手机浏览器当前代理出口 IP，不使用 Mac 的 `api.ipify` 结果，避免 BrowserScan 显示页面出口 IP 与 WebRTC STUN IP 不同。

## 运行命令

### 1. 本地单测

```bash
cd /Users/ethandong/code/nixiang/yuanlikeji/liulanqi
python3 generate_spoof_profile_test.py
python3 -m py_compile generate_spoof_profile.py
```

结果：`Ran 4 tests in 0.000s OK`。

### 2. 串行生成 3 组手机出口 IP 匹配 profile

```bash
cd /Users/ethandong/code/nixiang/yuanlikeji/liulanqi
OUT=.tmp/phone_ip_matched_multisite_20260427_batteryfix/generated
rm -rf "$OUT"
for i in 1 2 3; do
  mkdir -p "$OUT/p$i"
  python3 tools/remote/generate_phone_matched_spoof_profile.py \
    --adb-serial 5c8093e4 \
    --wait-seconds 6 \
    --brand Xiaomi \
    --model 2509FPN0BC \
    --webrtc-policy default_public_interface_only \
    --audio-mode random \
    --seed $((27042800 + i)) \
    --output-dir "$OUT/p$i"
done
```

每组生成时都通过手机浏览器访问 `https://api.ipify.org?format=json`，CDP 读到手机出口 IP：`84.245.9.218`。

### 3. 多检测网站回归

核心流程如下：

```bash
adb -s 5c8093e4 forward tcp:9223 localabstract:chrome_devtools_remote
adb -s 5c8093e4 push <profile.browser_profile.json> /data/local/tmp/profile_override_<label>_batteryfix.json
adb -s 5c8093e4 shell "printf '%s\n' '_ --fingerprint-profile-json=/data/local/tmp/profile_override_<label>_batteryfix.json --remote-allow-origins=*' > /data/local/tmp/chrome-command-line"
adb -s 5c8093e4 shell am start -S -W -n org.bromite.bromite/com.google.android.apps.chrome.Main -a android.intent.action.VIEW -d <检测网站 URL>
python/CDP Runtime.evaluate 读取 navigator、screen、WebGL、battery、mediaDevices、WebRTC candidates
```

检测网站：

- FingerprintJS: `https://fingerprintjs.github.io/fingerprintjs/`
- CreepJS: `https://abrahamjuliot.github.io/creepjs/`
- BrowserScan: `https://www.browserscan.net/`
- ToDetect: `https://www.todetect.net/`
- Pixelscan: `https://pixelscan.net/`
- AmIUnique: `https://amiunique.org/fingerprint`

### 4. FingerprintJS 精确组件检测

```bash
for p in .tmp/phone_ip_matched_multisite_20260427_batteryfix/generated/p*/*.browser_profile.json; do
  name=$(basename "$(dirname "$p")")
  python3 tools/remote/verify_fpjs_profile_on_phone.py \
    --adb-serial 5c8093e4 \
    --profile-path "$p" \
    --output-root ".tmp/phone_ip_matched_multisite_20260427_batteryfix/fpjs_exact_browser_profile/$name" \
    --skip-runtime-compare
done
```

## 生成 profile 数据

| 组 | profile_id | UA | screen | WebGL | battery | mediaDevices | WebRTC public IP |
|---|---|---|---|---|---|---|---|
| `p1` | `spoof-xiaomi-2509fpn0bc-1da9791187f8` | `Mozilla/5.0 (Linux; Android 16; 2509FPN0BC Build/BP2A.250605.031.A3; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/137.0.7151.115 Mobile Safari/537.36` | `400x870 DPR 3.0` | `Qualcomm / Adreno (TM) 840` | `{"level": 0.57, "charging": false, "chargingTime": 5536, "dischargingTime": 66680}` | `[{"kind": "audioinput", "deviceId": "", "groupId": "", "label": ""}, {"kind": "videoinput", "deviceId": "", "groupId": "", "label": ""}, {"kind": "audiooutput", "deviceId": "", "groupId": "", "label": ""}]` | `84.245.9.218` |
| `p2` | `spoof-xiaomi-2509fpn0bc-0735f1923088` | `Mozilla/5.0 (Linux; Android 16; 2509FPN0BC Build/BP2A.250605.031.A3; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/137.0.7151.115 Mobile Safari/537.36` | `400x870 DPR 3.0` | `Qualcomm / Adreno (TM) 840` | `{"level": 0.39, "charging": false, "chargingTime": 0, "dischargingTime": 66490}` | `[{"kind": "audioinput", "deviceId": "", "groupId": "", "label": ""}, {"kind": "videoinput", "deviceId": "", "groupId": "", "label": ""}, {"kind": "audiooutput", "deviceId": "", "groupId": "", "label": ""}]` | `84.245.9.218` |
| `p3` | `spoof-xiaomi-2509fpn0bc-3af17e5edfcc` | `Mozilla/5.0 (Linux; Android 16; 2509FPN0BC Build/BP2A.250605.031.A3; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/137.0.7151.115 Mobile Safari/537.36` | `400x870 DPR 3.0` | `Qualcomm / Adreno (TM) 840` | `{"level": 0.87, "charging": true, "chargingTime": 5024, "dischargingTime": 70700}` | `[{"kind": "audioinput", "deviceId": "", "groupId": "", "label": ""}, {"kind": "videoinput", "deviceId": "", "groupId": "", "label": ""}, {"kind": "audiooutput", "deviceId": "", "groupId": "", "label": ""}]` | `84.245.9.218` |

## 多站点检测结果总览

- 有效访问：`18/18`
- 稳定字段全一致：`18/18`
- 检查字段：UA、languages、UAData model、hardwareConcurrency、deviceMemory、screen、WebGL basics、WebGL extensions、WebGL numeric params、battery、mediaDevices。
- BrowserScan：`3/3` 未出现 `IP 地址不同`，页面正文 `3/3` 包含 STUN IP `84.245.9.218`。
- WebRTC candidate：`10/18` 次 CDP 主动 STUN 探针收敛到 `84.245.9.218`，未收敛的页面 candidate 数为 0；BrowserScan 页面自身 STUN 展示仍为 `84.245.9.218`。

| 组 | 网站 | allStable | UA | Lang | UAData | Screen | WebGL Basics | WebGL Exts | WebGL Params | Battery | Media | BrowserScan IP 不同 | BrowserScan STUN | Candidate 含 IP |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| `p1` | `fpjs` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `False` | `False` | `True` |
| `p1` | `creepjs` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `False` | `False` | `True` |
| `p1` | `browserscan` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `False` | `True` | `True` |
| `p1` | `todetect` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `False` | `False` | `False` |
| `p1` | `pixelscan` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `False` | `False` | `False` |
| `p1` | `amiunique` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `False` | `False` | `True` |
| `p2` | `fpjs` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `False` | `False` | `True` |
| `p2` | `creepjs` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `False` | `False` | `True` |
| `p2` | `browserscan` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `False` | `True` | `False` |
| `p2` | `todetect` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `False` | `False` | `False` |
| `p2` | `pixelscan` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `False` | `False` | `False` |
| `p2` | `amiunique` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `False` | `False` | `False` |
| `p3` | `fpjs` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `False` | `False` | `False` |
| `p3` | `creepjs` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `False` | `False` | `True` |
| `p3` | `browserscan` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `False` | `True` | `True` |
| `p3` | `todetect` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `False` | `False` | `False` |
| `p3` | `pixelscan` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `False` | `False` | `True` |
| `p3` | `amiunique` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `True` | `False` | `False` | `True` |

## BrowserScan 检测到数据样例

样例来自 `p1.browserscan.json`，profile 与页面实际检测值一致。

### Profile 生成值

```json
{
  "ua_string": "Mozilla/5.0 (Linux; Android 16; 2509FPN0BC Build/BP2A.250605.031.A3; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/137.0.7151.115 Mobile Safari/537.36",
  "languages": [
    "zh-CN",
    "en-US"
  ],
  "hardware_concurrency": 8,
  "device_memory_gb": 8,
  "screen_metrics": {
    "width": 400,
    "height": 870,
    "avail_width": 400,
    "avail_height": 870,
    "device_scale_factor": 3.0,
    "touch_supported": true,
    "color_depth": 24,
    "pixel_depth": 24
  },
  "webgl_basics": {
    "vendor": "WebKit",
    "version": "WebGL 1.0 (OpenGL ES 2.0 Chromium)",
    "renderer": "WebKit WebGL",
    "vendorUnmasked": "Qualcomm",
    "rendererUnmasked": "Adreno (TM) 840",
    "shadingLanguageVersion": "WebGL GLSL ES 1.0 (OpenGL ES GLSL ES 1.0 Chromium)"
  },
  "battery": {
    "level": 0.57,
    "charging": false,
    "chargingTime": 5536,
    "dischargingTime": 66680
  },
  "media_devices": [
    {
      "kind": "audioinput",
      "deviceId": "",
      "groupId": "",
      "label": ""
    },
    {
      "kind": "videoinput",
      "deviceId": "",
      "groupId": "",
      "label": ""
    },
    {
      "kind": "audiooutput",
      "deviceId": "",
      "groupId": "",
      "label": ""
    }
  ],
  "webrtc_public_ip": "84.245.9.218"
}
```

### BrowserScan 页面检测值

```json
{
  "userAgent": "Mozilla/5.0 (Linux; Android 16; 2509FPN0BC Build/BP2A.250605.031.A3; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/137.0.7151.115 Mobile Safari/537.36",
  "languages": [
    "zh-CN",
    "en-US"
  ],
  "hardwareConcurrency": 8,
  "deviceMemory": 8,
  "screen": {
    "width": 400,
    "height": 870,
    "availWidth": 400,
    "availHeight": 870,
    "devicePixelRatio": 3,
    "colorDepth": 24,
    "pixelDepth": 24
  },
  "webgl": {
    "vendor": "WebKit",
    "renderer": "WebKit WebGL",
    "unmaskedVendor": "Qualcomm",
    "unmaskedRenderer": "Adreno (TM) 840",
    "extensions": [
      "ANGLE_instanced_arrays",
      "EXT_blend_minmax",
      "EXT_color_buffer_half_float",
      "EXT_float_blend",
      "EXT_texture_compression_bptc",
      "EXT_texture_compression_rgtc",
      "EXT_texture_filter_anisotropic",
      "EXT_sRGB",
      "OES_element_index_uint",
      "OES_fbo_render_mipmap",
      "OES_standard_derivatives",
      "OES_texture_float",
      "OES_texture_float_linear",
      "OES_texture_half_float",
      "OES_texture_half_float_linear",
      "OES_vertex_array_object",
      "WEBGL_color_buffer_float",
      "WEBGL_compressed_texture_astc",
      "WEBGL_compressed_texture_etc",
      "WEBGL_compressed_texture_etc1",
      "WEBGL_compressed_texture_s3tc",
      "WEBGL_compressed_texture_s3tc_srgb",
      "WEBGL_debug_renderer_info",
      "WEBGL_debug_shaders",
      "WEBGL_depth_texture",
      "WEBGL_lose_context",
      "WEBGL_multi_draw"
    ],
    "parameters": {
      "MAX_TEXTURE_SIZE": 4096,
      "MAX_CUBE_MAP_TEXTURE_SIZE": 4096,
      "MAX_RENDERBUFFER_SIZE": 16384,
      "MAX_VIEWPORT_DIMS": [
        16384,
        16384
      ],
      "MAX_VERTEX_ATTRIBS": 32,
      "MAX_TEXTURE_IMAGE_UNITS": 16,
      "MAX_VERTEX_TEXTURE_IMAGE_UNITS": 16,
      "MAX_COMBINED_TEXTURE_IMAGE_UNITS": 96,
      "MAX_VERTEX_UNIFORM_VECTORS": 256,
      "MAX_FRAGMENT_UNIFORM_VECTORS": 256,
      "ALIASED_LINE_WIDTH_RANGE": [
        1,
        8
      ],
      "ALIASED_POINT_SIZE_RANGE": [
        1,
        1023
      ]
    }
  },
  "battery": {
    "level": 0.57,
    "charging": false,
    "chargingTime": 5536,
    "dischargingTime": 66680
  },
  "mediaDevices": [
    {
      "kind": "audioinput",
      "deviceId": "",
      "groupId": "",
      "label": ""
    },
    {
      "kind": "videoinput",
      "deviceId": "",
      "groupId": "",
      "label": ""
    },
    {
      "kind": "audiooutput",
      "deviceId": "",
      "groupId": "",
      "label": ""
    }
  ],
  "webrtcCandidates": [
    "candidate:1466078389 1 udp 1677729535 84.245.9.218 63374 typ srflx raddr 84.245.9.218 rport 0 generation 0 ufrag P/DA network-cost 999"
  ]
}
```

## WebGL numeric params 对照

| 参数 | profile 期望 | 页面检测实际 |
|---|---|---|
| `MAX_TEXTURE_SIZE` | `4096` | `4096` |
| `MAX_CUBE_MAP_TEXTURE_SIZE` | `4096` | `4096` |
| `MAX_RENDERBUFFER_SIZE` | `16384` | `16384` |
| `MAX_VIEWPORT_DIMS` | `[16384, 16384]` | `[16384, 16384]` |
| `MAX_VERTEX_ATTRIBS` | `32` | `32` |
| `MAX_TEXTURE_IMAGE_UNITS` | `16` | `16` |
| `MAX_VERTEX_TEXTURE_IMAGE_UNITS` | `16` | `16` |
| `MAX_COMBINED_TEXTURE_IMAGE_UNITS` | `96` | `96` |
| `MAX_VERTEX_UNIFORM_VECTORS` | `256` | `256` |
| `MAX_FRAGMENT_UNIFORM_VECTORS` | `256` | `256` |
| `ALIASED_LINE_WIDTH_RANGE` | `[1, 8]` | `[1, 8]` |
| `ALIASED_POINT_SIZE_RANGE` | `[1, 1023]` | `[1, 1023]` |

## FingerprintJS 精确组件结果

| 组 | canvas | fonts | domBlockers | audio | screenFrame | fontPreferences 页面输出 | audio target/actual | screenFrame target/actual |
|---|---|---|---|---|---|---|---|---|
| `p1` | `True` | `True` | `True` | `True` | `True` | `True` | `124.08075643484 / 124.08075643484` | `[0, 0, 0, 0] / [0, 0, 0, 0]` |
| `p2` | `True` | `True` | `True` | `True` | `True` | `True` | `124.08075643484 / 124.08075643484` | `[0, 0, 0, 0] / [0, 0, 0, 0]` |
| `p3` | `True` | `True` | `True` | `True` | `True` | `True` | `124.08075643484 / 124.08075643484` | `[0, 0, 0, 0] / [0, 0, 0, 0]` |

说明：`fontPreferencesRaw=false` 仅来自内部 raw 浮点的极小精度差，例如 `31.104167938232` vs `31.104167938232422`；FingerprintJS 页面最终输出值 `3/3` 一致。

## 原始结果文件

- 多站点 summary：`.tmp/phone_ip_matched_multisite_20260427_batteryfix/site_probe/summary.json`
- 多站点逐站 JSON：`.tmp/phone_ip_matched_multisite_20260427_batteryfix/site_probe/*.json`
- FPJS 精确组件 JSON：`.tmp/phone_ip_matched_multisite_20260427_batteryfix/fpjs_exact_browser_profile/*/record_*/verify_result.json`
- 生成 profile：`.tmp/phone_ip_matched_multisite_20260427_batteryfix/generated/p*/*.browser_profile.json`
