---
name: 国内官方 APK 下载与验真
description: >-
  【国内官方 APK 下载与验真】>- 【国内官方 APK 下载与验真】从国内渠道下载「官方正版」安卓 APK（手机/电视通用），并做包名/签名/MD5 三重校验，识别山寨包。当需要给安卓手机、小米/海信等智能电视、电视盒子、U 盘装机准备 APK 时使用。触发词：下载 apk、装电视、U盘安装、红果/抖音/番茄类国内 App 安装包、电视版 apk。
agent_created: true
---

# 国内官方 APK 下载与验真

## 核心结论（省时间，直接抄）

**唯一稳定好用的直链通道 = 豌豆荚。**

```
详情页:  https://www.wandoujia.com/apps/<包名>
下载:    https://www.wandoujia.com/apps/<数字appid>/download
        → 302 → https://ucdl.25pp.com/fs08/<日期>/<n>/<hash>.apk?...&md5=<32位MD5>
```

- 数字 appid 从详情页 HTML 里取：`href="https://www.wandoujia.com/apps/8396801/download/..."`
- 302 的 Location 里带 `fname=`（中文名）、`pkg=`（包名）、`vcode=`（版本码）、`md5=`（官方 MD5）→ **下载完必须比对 md5**
- CDN 实测 15–18 MB/s，128MB 的包 7 秒下完，无需分段

## 各渠道实测状态（2026-09）

| 渠道 | 状态 | 说明 |
|---|---|---|
| 豌豆荚 wandoujia.com | ✅ 可用 | 服务端 302，直链最干净，首选 |
| 应用宝 sj.qq.com / android.myapp.com | ❌ | SPA，HTML 里只有腾讯管家自己的 apk，需 CDP 抓 XHR |
| 小米应用商店 app.mi.com | ❌ | 详情页是 JS 渲染；`/download/<id>` 会 302 回详情页，拿不到真直链 |
| APKPure | ❌ | 本机连不上（WinError 10060） |
| `hongguo.heytapmobi.com` 之类"官网" | ❌ 假的 | DNS 直接解析失败；多个内容站互相抄这个域名当"官网"，纯 SEO 农场 |
| 3DM/7po/mfunz 等下载站 | ⚠️ | 页面异步，直链要 JS；且混山寨包 |

## 验真三步（缺一不可）

1. **包名**：从 `AndroidManifest.xml` 里找。注意是 **UTF-16 字符串池**，要这么找：
   ```python
   z.read('AndroidManifest.xml').count('com.phoenix.read'.encode('utf-16-le'))  # >0 即命中
   ```
   用 ASCII 的 `b'com.phoenix.read'` 去 find 会得到 0，别被误导。
2. **签名**：`META-INF/` 下看签名文件名。
   - `BYTESIGN.RSA` / `BYTESIGN.SF` → **字节跳动系**（抖音/番茄/红果）；`.SF` 里 `X-Android-APK-Signed: 2, 3` 说明有 v2+v3 签名
   - 再用 openssl 解出主体，确认落地实体：
     ```bash
     openssl pkcs7 -inform DER -in META-INF/BYTESIGN.RSA -print_certs -noout
     # 实测红果: subject=C=86, ST=gd, L=sz, O=fq, OU=fq, CN=novel  (深圳·番茄)
     ```
3. **MD5**：与直链里 `md5=` 参数比对。一致 = 官方源原包。

## 山寨包识别

- **包名 `com.example.*` 一律是套壳**。实测"红果TV电视版"包名 `com.example.jwzt_jincheng`，21MB（正版 128MB），却盗用"北京笔墨留香科技有限公司"当厂商名 —— 典型的破包。
- 体积异常小 / 版本号与官网对不上 / 要求通讯录·短信权限 → 直接丢。
- 网上"XX TV版""XX去广告版""破解版"基本都是手机版改名或套壳。

## 电视端的真相（重要，别被搜索结果骗）

**大多数国内短剧/短视频 App 没有独立 TV 版 APK。** 搜索结果里那些"红果短剧TV版 v7.2.4 电视版"就是把手机版 APK 换个标题。

给电视装的正确姿势，按优先级：
1. **电视自带应用商店搜**（小米/海信/创维的商店可能已上架，一键装，适配最好）
2. **U 盘装手机版 APK**（能装能跑，但界面竖屏、遥控器操作别扭，建议配无线鼠标）
3. **手机投屏**（DLNA/镜像，画质体验最好，零安装）

装完记得提醒用户：电视端"允许安装未知来源应用"要开（设置→安全）。

## 【高频坑】装电视报"版本不兼容" = ABI 不匹配

**现象**：U 盘装 APK，电视提示"版本不兼容 / 与设备不兼容"。

**先别换包，先查 ABI。** 手机版 App 极可能只有 64 位库，而电视（小米 4A/4C/4S、Amlogic T962/T968 等）多为 **32 位 Android**，只认 `armeabi-v7a`。

### 极速 ABI 探测（不用下整个包！）

ZIP 的文件列表在**文件末尾**，用 HTTP Range 只拉尾部即可：

1. `Range: bytes=-65536` 拉末尾 64KB → 找 `PK\x05\x06` (EOCD)
2. EOCD 偏移 +12 读 `cd_size`，+16 读 `cd_off`
3. `Range: bytes=<cd_off>-<cd_off+cd_size-1>` 拉中央目录
4. 遍历 `PK\x01\x02` 条目，收集 `lib/<abi>/` 里的 abi

2 次请求、约 2MB 流量就能判定架构，比下 130MB 快 60 倍。

### 实测结论（2026-09）

**字节系 App（红果 `com.phoenix.read`、番茄等）从 2024 到 2026 全系列只发 `arm64-v8a`**，扫 v6.2.3.32 / v6.4.9.32 / v6.7.7.32 / v7.0.5.32 / v7.2.9.32 / v7.3.5.32 一致。**32 位电视永远装不了红果手机版**，换版本没用。

### 32 位电视的正确解法

1. **电视自带应用商店搜**（小米/海信等按机型下发适配版本）—— 成功率最高
2. **U 盘装「当贝市场」再从里面装** —— 当贝是 TV 专用市场，收录的就是 TV 版
3. **手机投屏** —— 绕开架构问题，体验反而最好

**当贝市场 TV 版官方直链**（已实测，32/64 位通吃）：
```
https://webapk.dangbei.net/update/dbmarket.apk
包名 com.dangbeimarket，ABI = arm64-v8a + armeabi + armeabi-v7a
```

### 拿旧版本的最省事入口

豌豆荚历史版本页**直接内嵌 APK 直链**，不用点 JS：
```
https://www.wandoujia.com/apps/<appid>/history_v<版本码>
→ 页面里 https://android-apps.pp.cn/.../<hash>.apk?...&md5=..&minSDK=..&size=..
```
版本码 = 版本号去掉点（7.0.5.32 → 70532）。首页 `history` 列出全部版本链接，含年份页 `history_y2024` / `history_y2023`。

## 踩坑

- **U 盘写入**：Bash/Python 直接写 `G:\` 可用（U 盘是真实块设备，不受工作区沙箱限制，实测成功）。工作区外的固定盘（如 `D:\Tools`）才会丢。
- 文件名用中文没问题，`os.replace(tmp, OUT)` 先写 `.part` 再改名，避免半截文件被电视识别。
- 别用 `wmic`（已下线），也别用 Bash 调 powershell（被安全策略拦）。查盘用 `ls /g/` + `df -h /g` 就够。
