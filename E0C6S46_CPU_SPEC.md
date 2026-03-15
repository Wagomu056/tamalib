# Epson E0C6S46 / E0C6S48 CPU 仕様書

TamaLIB の実装 (`tamalib/cpu.c`, `tamalib/hw.c`) を解析したものです。
初代たまごっちに搭載されている 4bit RISC CPU の仕様をまとめています。

---

## 目次

1. [アーキテクチャ概要](#1-アーキテクチャ概要)
2. [レジスタ](#2-レジスタ)
3. [メモリマップ](#3-メモリマップ)
4. [ROMフォーマットと配置](#4-romフォーマットと配置)
5. [命令セット](#5-命令セット)
6. [フラグ](#6-フラグ)
7. [LCD ディスプレイ](#7-lcd-ディスプレイ)
8. [I/O レジスタ一覧](#8-io-レジスタ一覧)
9. [割り込みシステム](#9-割り込みシステム)
10. [タイマー](#10-タイマー)
11. [リセット後の初期状態](#11-リセット後の初期状態)
12. [ROM 開発チートシート](#12-rom-開発チートシート)

---

## 1. アーキテクチャ概要

| 項目 | E0C6S46 (P1) | E0C6S48 (P2, Angel) |
|------|-------------|---------------------|
| データ幅 | 4 bit | 4 bit |
| 命令幅 | 12 bit | 12 bit |
| PCアドレス幅 | 13 bit (最大 8192命令) | 13 bit |
| RAM | 640 nibble (0x000–0x27F) | 768 nibble (0x000–0x2FF) |
| ディスプレイメモリ | 80 nibble × 2バンク | 102 nibble × 2バンク |
| OSCクロック (OSC1) | 32,768 Hz | 32,768 Hz |
| OSCクロック (OSC3) | — | 1,000,000 Hz |

---

## 2. レジスタ

### 汎用レジスタ

| レジスタ | ビット幅 | 説明 |
|---------|---------|------|
| `A` | 4 bit | アキュムレータ A |
| `B` | 4 bit | アキュムレータ B |
| `X` | 12 bit | インデックスレジスタ X (XP:XH:XL の3ニブル構成) |
| `Y` | 12 bit | インデックスレジスタ Y (YP:YH:YL の3ニブル構成) |
| `SP` | 8 bit | スタックポインタ (SPH:SPL の2ニブル構成) |
| `NP` | 5 bit | コードページレジスタ (NBP:NPP) |
| `PC` | 13 bit | プログラムカウンタ (PCB:PCP:PCSH:PCSL) |
| `F` | 4 bit | フラグレジスタ (C, Z, D, I) |

### X / Y レジスタの内部構成

```
X[11:8] = XP  (ページ部: 0x0–0xF)
X[7:4]  = XH  (上位ニブル)
X[3:0]  = XL  (下位ニブル)
X[7:0]  = XHL (LD X #imm で設定できる範囲)
```

同様に Y: YP, YH, YL, YHL。

### PC の内部構成

```
PC[12]   = PCB  (バンクビット)
PC[11:8] = PCP  (ページ番号 0x0–0xF)
PC[7:4]  = PCSH (ステップ上位ニブル)
PC[3:0]  = PCSL (ステップ下位ニブル)
PC[7:0]  = PCS  (ステップ: 0x00–0xFF)
```

### NP の内部構成

```
NP[4]   = NBP  (バンクビット)
NP[3:0] = NPP  (ページ番号)
```

### レジスタ選択 (R/Q フィールド)

多くの命令はオペランドに 2bit の "R(x)" / "Q(x)" フィールドを持つ。

| 値 | 意味 |
|----|------|
| 0  | A |
| 1  | B |
| 2  | M(X) ← X が指すメモリ |
| 3  | M(Y) ← Y が指すメモリ |

---

## 3. メモリマップ

### E0C6S46 (P1 = 初代たまごっち)

| アドレス | サイズ | 用途 |
|---------|--------|------|
| `0x000–0x27F` | 640 nibble | RAM |
| `0x280–0xDFF` | — | 無効 |
| `0xE00–0xE4F` | 80 nibble | ディスプレイメモリ 1 |
| `0xE50–0xE7F` | — | 無効 |
| `0xE80–0xECF` | 80 nibble | ディスプレイメモリ 2 |
| `0xED0–0xEFF` | — | 無効 |
| `0xF00–0xF7F` | 128 nibble | I/O レジスタ |

### E0C6S48 (P2, Angel)

| アドレス | サイズ | 用途 |
|---------|--------|------|
| `0x000–0x2FF` | 768 nibble | RAM |
| `0x300–0xDFF` | — | 無効 |
| `0xE00–0xE65` | 102 nibble | ディスプレイメモリ 1 |
| `0xE66–0xE7F` | — | 無効 |
| `0xE80–0xEE5` | 102 nibble | ディスプレイメモリ 2 |
| `0xEE6–0xEFF` | — | 無効 |
| `0xF00–0xF7F` | 128 nibble | I/O レジスタ |

> **注意**: すべてのアドレスは 4bit ニブル単位のアドレスです。
> 物理的なバイト数はこれの半分 (2ニブル = 1バイト) です。

---

## 4. ROMフォーマットと配置

### バイナリフォーマット

各命令は **2バイト** でエンコードされる。

```
ROM byte[0] & 0x0F = 命令の bits[11:8] (上位4bit)
ROM byte[1]        = 命令の bits[7:0]  (下位8bit)
```

エンコード/デコード式:
```python
# エンコード (12bit命令 → 2バイト)
def encode(op12):
    return bytes([(op12 >> 8) & 0xF, op12 & 0xFF])

# デコード (2バイト → 12bit命令)
def decode(b0, b1):
    return b1 | ((b0 & 0xF) << 8)
```

### ファイルサイズ

標準 ROM サイズ: **12,288 バイト** (= 6,144 命令)

### 命令アドレスとバイトオフセットの関係

```
バイトオフセット = 命令アドレス (PC値) × 2
```

例:
- PC = `0x0000` → バイトオフセット `0x0000`
- PC = `0x0100` → バイトオフセット `0x0200`

### リセット後のエントリポイント

```
PC = 0x0100   (bank=0, page=1, step=0x00)
NP = 0x01     (bank=0, page=1)
```

**プログラムは ROM のバイトオフセット `0x0200` (= PC `0x0100`) から始める必要がある。**

### 無効領域のパディング

ROM の未使用領域を `0x00` で埋めると、`0x000` = `JP #0x00` (ページ0の先頭へジャンプ) として解釈される。通常は無害。

---

## 5. 命令セット

### 命令フォーマット表記

- `#imm` : 即値
- `R(x)` : レジスタ選択 (A/B/MX/MY)
- `Q(x)` : レジスタ選択 (同上)
- `M(n)` : アドレス n のメモリ
- `C` : キャリーフラグ

### マスク定義

| 定数 | 値 | 説明 |
|------|----|------|
| MASK_4B | `0xF00` | 上位4bit固定 → 8bit即値 |
| MASK_6B | `0xFC0` | 上位6bit固定 → 6bit可変 |
| MASK_7B | `0xFE0` | 上位7bit固定 → 5bit可変 |
| MASK_8B | `0xFF0` | 上位8bit固定 → 4bit即値 |
| MASK_10B | `0xFFC` | 上位10bit固定 → 2bit可変 |
| MASK_12B | `0xFFF` | 全bit固定 (引数なし) |

---

### 5.1 制御フロー命令

| ニーモニック | オペコード | マスク | サイクル | 動作 |
|------------|----------|--------|---------|------|
| `JP #imm` | `0x000` | MASK_4B | 5 | `PC = imm \| (NP << 8)` |
| `JP C #imm` | `0x200` | MASK_4B | 5 | C=1 なら `PC = imm \| (NP << 8)` |
| `JP NC #imm` | `0x300` | MASK_4B | 5 | C=0 なら同上 |
| `JP Z #imm` | `0x600` | MASK_4B | 5 | Z=1 なら同上 |
| `JP NZ #imm` | `0x700` | MASK_4B | 5 | Z=0 なら同上 |
| `JPBA` | `0xFE8` | MASK_12B | 5 | `PC = A \| (B << 4) \| (NP << 8)` |
| `CALL #imm` | `0x400` | MASK_4B | 7 | スタックに PC+1 を退避、`PC = PCB:NPP:imm` |
| `CALZ #imm` | `0x500` | MASK_4B | 7 | スタックに PC+1 を退避、`PC = PCB:0:imm` |
| `RET` | `0xFDF` | MASK_12B | 7 | スタックから PC を復帰 |
| `RETS` | `0xFDE` | MASK_12B | 12 | スタックから PC を復帰して +1 スキップ |
| `RETD #imm` | `0x100` | MASK_4B | 12 | RET + M[X],M[X+1] に imm(8bit) を書いて X+=2 |
| `PSET #imm` | `0xE40` | MASK_7B | 5 | `NP = imm` (5bit: コードページ設定) |
| `NOP5` | `0xFFB` | MASK_12B | 5 | 何もしない |
| `NOP7` | `0xFFF` | MASK_12B | 7 | 何もしない |
| `HALT` | `0xFF8` | MASK_12B | 5 | CPU停止 (割り込みで復帰) |

> **NP の自動リセット**: PSET を除くすべての命令の実行後、NP は自動的に現在の PC のページ/バンクにリセットされる (`NP = (PC >> 8) & 0x1F`)。
> そのため、PSET で設定した NP は **直後の1命令** (通常は JP または CALL) でのみ有効。
> PSET と JP/CALL の間に他の命令を挟むと NP がリセットされ、意図したページにジャンプできない。

> **JP の注意点**: `JP #imm` のジャンプ先は `imm | (NP << 8)`。
> リセット直後は NP=1 なので、`JP #0x04` → PC = `0x104`。
> 別ページに飛ぶには先に `PSET` で NP を設定すること。

> **CALL / CALZ の違い**: `CALL` は NP の現在ページを使う。`CALZ` は常にページ0を使う。

---

### 5.2 ロード命令

| ニーモニック | オペコード | マスク | サイクル | 動作 |
|------------|----------|--------|---------|------|
| `LD R(x) #imm` | `0xE00` | MASK_6B | 5 | `R(x) = imm` (4bit) |
| `LD R(x) Q(x)` | `0xEC0` | MASK_8B | 5 | `R(x) = Q(x)` |
| `LD A M(#n)` | `0xFA0` | MASK_8B | 5 | `A = M(n)` (n は下位4bitで0x0–0xF) |
| `LD B M(#n)` | `0xFB0` | MASK_8B | 5 | `B = M(n)` |
| `LD M(#n) A` | `0xF80` | MASK_8B | 5 | `M(n) = A` |
| `LD M(#n) B` | `0xF90` | MASK_8B | 5 | `M(n) = B` |
| `LD X #imm` | `0xB00` | MASK_4B | 5 | `X[7:0] = imm` (XP は変更しない) |
| `LD Y #imm` | `0x800` | MASK_4B | 5 | `Y[7:0] = imm` (YP は変更しない) |
| `LD XP R(x)` | `0xE80` | MASK_10B | 5 | `XP = R(x)` |
| `LD XH R(x)` | `0xE84` | MASK_10B | 5 | `XH = R(x)` |
| `LD XL R(x)` | `0xE88` | MASK_10B | 5 | `XL = R(x)` |
| `LD YP R(x)` | `0xE90` | MASK_10B | 5 | `YP = R(x)` |
| `LD YH R(x)` | `0xE94` | MASK_10B | 5 | `YH = R(x)` |
| `LD YL R(x)` | `0xE98` | MASK_10B | 5 | `YL = R(x)` |
| `LD R(x) XP` | `0xEA0` | MASK_10B | 5 | `R(x) = XP` |
| `LD R(x) XH` | `0xEA4` | MASK_10B | 5 | `R(x) = XH` |
| `LD R(x) XL` | `0xEA8` | MASK_10B | 5 | `R(x) = XL` |
| `LD R(x) YP` | `0xEB0` | MASK_10B | 5 | `R(x) = YP` |
| `LD R(x) YH` | `0xEB4` | MASK_10B | 5 | `R(x) = YH` |
| `LD R(x) YL` | `0xEB8` | MASK_10B | 5 | `R(x) = YL` |
| `LD SPH R(x)` | `0xFE0` | MASK_10B | 5 | `SPH = R(x)` |
| `LD SPL R(x)` | `0xFF0` | MASK_10B | 5 | `SPL = R(x)` |
| `LD R(x) SPH` | `0xFE4` | MASK_10B | 5 | `R(x) = SPH` |
| `LD R(x) SPL` | `0xFF4` | MASK_10B | 5 | `R(x) = SPL` |
| `LDPX MX #imm` | `0xE60` | MASK_8B | 5 | `M[X] = imm; X++` (XP は変えない) |
| `LDPX R(x) Q(x)` | `0xEE0` | MASK_8B | 5 | `R(x) = Q(x); X++` ※R=0,Q=0 (=`0xEE0`) は INC X として扱われる |
| `LDPY MY #imm` | `0xE70` | MASK_8B | 5 | `M[Y] = imm; Y++` (YP は変えない) |
| `LDPY R(x) Q(x)` | `0xEF0` | MASK_8B | 5 | `R(x) = Q(x); Y++` ※R=0,Q=0 (=`0xEF0`) は INC Y として扱われる |
| `LBPX #imm` | `0x900` | MASK_4B | 5 | `M[X]=imm[3:0]; M[X+1]=imm[7:4]; X+=2` (8bit一括書き込み) |
| `INC X` | `0xEE0` | MASK_12B | 5 | `X = (X+1) & 0xFF \| (XP<<8)` ※LDPX A,A と同一オペコードだが命令テーブルで先にマッチする |
| `INC Y` | `0xEF0` | MASK_12B | 5 | `Y = (Y+1) & 0xFF \| (YP<<8)` ※LDPY A,A と同一オペコードだが命令テーブルで先にマッチする |

---

### 5.3 スタック命令

| ニーモニック | オペコード | マスク | サイクル | 動作 |
|------------|----------|--------|---------|------|
| `PUSH R(x)` | `0xFC0` | MASK_10B | 5 | `M[--SP] = R(x)` |
| `PUSH XP` | `0xFC4` | MASK_12B | 5 | `M[--SP] = XP` |
| `PUSH XH` | `0xFC5` | MASK_12B | 5 | `M[--SP] = XH` |
| `PUSH XL` | `0xFC6` | MASK_12B | 5 | `M[--SP] = XL` |
| `PUSH YP` | `0xFC7` | MASK_12B | 5 | `M[--SP] = YP` |
| `PUSH YH` | `0xFC8` | MASK_12B | 5 | `M[--SP] = YH` |
| `PUSH YL` | `0xFC9` | MASK_12B | 5 | `M[--SP] = YL` |
| `PUSH F` | `0xFCA` | MASK_12B | 5 | `M[--SP] = flags` |
| `POP R(x)` | `0xFD0` | MASK_10B | 5 | `R(x) = M[SP++]` |
| `POP XP` | `0xFD4` | MASK_12B | 5 | `XP = M[SP++]` |
| `POP XH` | `0xFD5` | MASK_12B | 5 | `XH = M[SP++]` |
| `POP XL` | `0xFD6` | MASK_12B | 5 | `XL = M[SP++]` |
| `POP YP` | `0xFD7` | MASK_12B | 5 | `YP = M[SP++]` |
| `POP YH` | `0xFD8` | MASK_12B | 5 | `YH = M[SP++]` |
| `POP YL` | `0xFD9` | MASK_12B | 5 | `YL = M[SP++]` |
| `POP F` | `0xFDA` | MASK_12B | 5 | `flags = M[SP++]` |
| `INC SP` | `0xFDB` | MASK_12B | 5 | `SP++` |
| `DEC SP` | `0xFCB` | MASK_12B | 5 | `SP--` |

---

### 5.4 算術命令

| ニーモニック | オペコード | マスク | サイクル | 動作 |
|------------|----------|--------|---------|------|
| `ADD R(x) #imm` | `0xC00` | MASK_6B | 7 | `R(x) += imm` → C, Z 更新 |
| `ADD R(x) Q(x)` | `0xA80` | MASK_8B | 7 | `R(x) += Q(x)` → C, Z 更新 |
| `ADC R(x) #imm` | `0xC40` | MASK_6B | 7 | `R(x) += imm + C` → C, Z 更新 |
| `ADC R(x) Q(x)` | `0xA90` | MASK_8B | 7 | `R(x) += Q(x) + C` → C, Z 更新 |
| `ADC XH #imm` | `0xA00` | MASK_8B | 7 | `XH += imm + C` → C, Z 更新 |
| `ADC XL #imm` | `0xA10` | MASK_8B | 7 | `XL += imm + C` → C, Z 更新 |
| `ADC YH #imm` | `0xA20` | MASK_8B | 7 | `YH += imm + C` → C, Z 更新 |
| `ADC YL #imm` | `0xA30` | MASK_8B | 7 | `YL += imm + C` → C, Z 更新 |
| `SUB R(x) Q(x)` | `0xAA0` | MASK_8B | 7 | `R(x) -= Q(x)` → C, Z 更新 |
| `SBC R(x) #imm` | `0xD40` | MASK_6B | 7 | `R(x) -= imm + C` → C, Z 更新 |
| `SBC R(x) Q(x)` | `0xAB0` | MASK_8B | 7 | `R(x) -= Q(x) + C` → C, Z 更新 |
| `INC M(#n)` | `0xF60` | MASK_8B | 7 | `M(n)++` → C, Z 更新 |
| `DEC M(#n)` | `0xF70` | MASK_8B | 7 | `M(n)--` → C, Z 更新 |
| `ACPX R(x)` | `0xF28` | MASK_10B | 7 | `M[X] += R(x) + C; X++` → C, Z 更新 |
| `ACPY R(x)` | `0xF2C` | MASK_10B | 7 | `M[Y] += R(x) + C; Y++` → C, Z 更新 |
| `SCPX R(x)` | `0xF38` | MASK_10B | 7 | `M[X] -= R(x) + C; X++` → C, Z 更新 |
| `SCPY R(x)` | `0xF3C` | MASK_10B | 7 | `M[Y] -= R(x) + C; Y++` → C, Z 更新 |

> D フラグが立っているとき加減算は BCD モードで動作する。

---

### 5.5 論理命令

| ニーモニック | オペコード | マスク | サイクル | 動作 |
|------------|----------|--------|---------|------|
| `AND R(x) #imm` | `0xC80` | MASK_6B | 7 | `R(x) &= imm` → Z 更新 |
| `AND R(x) Q(x)` | `0xAC0` | MASK_8B | 7 | `R(x) &= Q(x)` → Z 更新 |
| `OR R(x) #imm` | `0xCC0` | MASK_6B | 7 | `R(x) \|= imm` → Z 更新 |
| `OR R(x) Q(x)` | `0xAD0` | MASK_8B | 7 | `R(x) \|= Q(x)` → Z 更新 |
| `XOR R(x) #imm` | `0xD00` | MASK_6B | 7 | `R(x) ^= imm` → Z 更新 |
| `XOR R(x) Q(x)` | `0xAE0` | MASK_8B | 7 | `R(x) ^= Q(x)` → Z 更新 |
| `NOT R(x)` | `0xD0F` | `0xFCF` | 7 | `R(x) = ~R(x) & 0xF` → Z 更新 |
| `RLC R(x)` | `0xAF0` | MASK_8B | 7 | 左ローテート through C (C←bit3, bit0←旧C) |
| `RRC R(x)` | `0xE8C` | MASK_10B | 5 | 右ローテート through C (C←bit0, bit3←旧C) |

---

### 5.6 比較命令

| ニーモニック | オペコード | マスク | サイクル | 動作 |
|------------|----------|--------|---------|------|
| `CP R(x) #imm` | `0xDC0` | MASK_6B | 7 | `R(x) < imm` → C=1; `==` → Z=1 |
| `CP R(x) Q(x)` | `0xF00` | MASK_8B | 7 | `R(x) < Q(x)` → C=1; `==` → Z=1 |
| `CP XH #imm` | `0xA40` | MASK_8B | 7 | XH と比較 |
| `CP XL #imm` | `0xA50` | MASK_8B | 7 | XL と比較 |
| `CP YH #imm` | `0xA60` | MASK_8B | 7 | YH と比較 |
| `CP YL #imm` | `0xA70` | MASK_8B | 7 | YL と比較 |
| `FAN R(x) #imm` | `0xD80` | MASK_6B | 7 | `R(x) & imm == 0` → Z=1 (AND test, 値は変更しない) |
| `FAN R(x) Q(x)` | `0xF10` | MASK_8B | 7 | `R(x) & Q(x) == 0` → Z=1 |

---

### 5.7 フラグ操作命令

| ニーモニック | オペコード | マスク | サイクル | 動作 |
|------------|----------|--------|---------|------|
| `SCF` | `0xF41` | MASK_12B | 7 | C = 1 |
| `RCF` | `0xF5E` | MASK_12B | 7 | C = 0 |
| `SZF` | `0xF42` | MASK_12B | 7 | Z = 1 |
| `RZF` | `0xF5D` | MASK_12B | 7 | Z = 0 |
| `SDF` | `0xF44` | MASK_12B | 7 | D = 1 (BCDモード有効) |
| `RDF` | `0xF5B` | MASK_12B | 7 | D = 0 |
| `EI` | `0xF48` | MASK_12B | 7 | I = 1 (割り込み有効) |
| `DI` | `0xF57` | MASK_12B | 7 | I = 0 (割り込み無効) |
| `SET #imm` | `0xF40` | MASK_8B | 7 | `flags \|= imm` |
| `RST #imm` | `0xF50` | MASK_8B | 7 | `flags &= imm` |

---

## 6. フラグ

| ビット | 名前 | 説明 |
|-------|------|------|
| bit 0 | C (Carry) | 桁上がり/借り、シフト時のあふれビット |
| bit 1 | Z (Zero) | 演算結果が 0 のとき 1 |
| bit 2 | D (Decimal) | 1 = BCD算術モード |
| bit 3 | I (Interrupt) | 1 = 割り込み有効 |

算術/論理命令後のフラグ更新規則:
- ADD/ADC/SUB/SBC → C と Z を更新
- AND/OR/XOR/NOT → Z のみ更新
- CP → C と Z を更新 (値は変更しない)
- FAN → Z のみ更新 (値は変更しない)
- RLC/RRC → C のみ更新 (Z は更新しない)

---

## 7. LCD ディスプレイ

### ディスプレイ構成

| 項目 | 値 |
|------|-----|
| 解像度 | 32 × 16 ピクセル |
| アイコン | 8 個 |
| 色 | モノクロ (1bit/pixel) |

### ディスプレイメモリとピクセルの対応

ディスプレイメモリ (0xE00–) への書き込みは即座に LCD に反映される。

アドレス `n` に値 `v` を書き込んだときの動作:
```
seg  = (n & 0x7F) >> 1
com0 = ((n & 0x80) >> 7) * 8 + (n & 0x1) * 4

v の各ビット i (0–3) が:
  LCD pixel (seg_pos[seg], com0 + i) に対応
```

SEG と LCD X 座標の対応 (seg_pos テーブル):
```
SEG:  0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19 ...
LCD: [0, 1, 2, 3, 4, 5, 6, 7,32, 8, 9,10,11,12,13,14,15,33,34,35...]
```
(SEG 8, 28 の一部は LCD アイコンに使用)

### ディスプレイメモリ 1 と 2 の違い

| バンク | アドレス範囲 | com0 の計算 |
|--------|------------|-------------|
| DISP1 | `0xE00–0xE4F` | `(n & 0x1) * 4` → COM 0–7 |
| DISP2 | `0xE80–0xECF` | `8 + (n & 0x1) * 4` → COM 8–15 |

### よく使うアドレスとピクセル位置

| アドレス | seg | com0 | ピクセル範囲 |
|---------|-----|------|------------|
| `0xE00` | 0 | 0 | (x=0, y=0–3) |
| `0xE01` | 0 | 4 | (x=0, y=4–7) |
| `0xE02` | 1 | 0 | (x=1, y=0–3) |
| `0xE80` | 0 | 8 | (x=0, y=8–11) |
| `0xE81` | 0 | 12 | (x=0, y=12–15) |

**例**: `M[0xE00] = 0x1` → seg=0, com=0 のピクセル 1個が点灯 → LCD の左上 (0, 0) が点灯

**例**: `M[0xE00] = 0xF` → seg=0, com=0–3 の 4個が点灯 → x=0 の上4ピクセルが点灯

### アイコン

| SEG | COM | アイコン番号 |
|-----|-----|------------|
| 8 | 0 | icon 0 |
| 8 | 1 | icon 1 |
| 8 | 2 | icon 2 |
| 8 | 3 | icon 3 |
| 28 | 12 | icon 4 |
| 28 | 13 | icon 5 |
| 28 | 14 | icon 6 |
| 28 | 15 | icon 7 |

---

## 8. I/O レジスタ一覧

| アドレス | 定数名 | 用途 |
|---------|--------|------|
| `0xF00` | REG_CLK_INT_FACTOR_FLAGS | クロック割り込みフラグ |
| `0xF01` | REG_SW_INT_FACTOR_FLAGS | ストップウォッチ割り込みフラグ |
| `0xF02` | REG_PROG_INT_FACTOR_FLAGS | プログラマブルタイマー割り込みフラグ |
| `0xF03` | REG_SERIAL_INT_FACTOR_FLAGS | シリアル割り込みフラグ |
| `0xF04` | REG_K00_K03_INT_FACTOR_FLAGS | K00–K03 入力割り込みフラグ |
| `0xF05` | REG_K10_K13_INT_FACTOR_FLAGS | K10–K13 入力割り込みフラグ |
| `0xF10` | REG_CLOCK_INT_MASKS | クロック割り込みマスク |
| `0xF11` | REG_SW_INT_MASKS | ストップウォッチ割り込みマスク |
| `0xF12` | REG_PROG_INT_MASKS | プログラマブルタイマー割り込みマスク |
| `0xF13` | REG_SERIAL_INT_MASKS | シリアル割り込みマスク |
| `0xF14` | REG_K00_K03_INT_MASKS | K00–K03 入力割り込みマスク |
| `0xF15` | REG_K10_K13_INT_MASKS | K10–K13 入力割り込みマスク |
| `0xF20` | REG_CLOCK_TIMER_DATA_1 | クロックタイマーデータ 1 |
| `0xF21` | REG_CLOCK_TIMER_DATA_2 | クロックタイマーデータ 2 |
| `0xF22` | REG_SW_TIMER_DATA_L | ストップウォッチデータ L |
| `0xF23` | REG_SW_TIMER_DATA_H | ストップウォッチデータ H |
| `0xF24` | REG_PROG_TIMER_DATA_L | プログラマブルタイマーデータ L |
| `0xF25` | REG_PROG_TIMER_DATA_H | プログラマブルタイマーデータ H |
| `0xF26` | REG_PROG_TIMER_RELOAD_DATA_L | プログラマブルタイマーリロードデータ L |
| `0xF27` | REG_PROG_TIMER_RELOAD_DATA_H | プログラマブルタイマーリロードデータ H |
| `0xF30` | REG_SERIAL_IF_DATA_L | シリアルインターフェースデータ L |
| `0xF31` | REG_SERIAL_IF_DATA_H | シリアルインターフェースデータ H |
| `0xF40` | REG_K00_K03_INPUT_PORT | K00–K03 入力ポート |
| `0xF41` | REG_K00_K03_INPUT_RELATION | K00–K03 アクティブレベル設定 (初期値 0xF = Active HIGH) |
| `0xF42` | REG_K10_K13_INPUT_PORT | K10–K13 入力ポート |
| `0xF50` | REG_R00_R03_OUTPUT_PORT | R00–R03 出力ポート |
| `0xF51` | REG_R10_R13_OUTPUT_PORT | R10–R13 出力ポート |
| `0xF52` | REG_R20_R23_OUTPUT_PORT | R20–R23 出力ポート |
| `0xF53` | REG_R30_R33_OUTPUT_PORT | R30–R33 出力ポート |
| `0xF54` | REG_R40_R43_BZ_OUTPUT_PORT | R40–R43/ブザー出力ポート (初期値 0xF) |
| `0xF60` | REG_P00_P03_IO_PORT | P00–P03 I/O ポート |
| `0xF61` | REG_P10_P13_IO_PORT | P10–P13 I/O ポート |
| `0xF62` | REG_P20_P23_IO_PORT | P20–P23 I/O ポート |
| `0xF63` | REG_P30_P33_IO_PORT | P30–P33 I/O ポート |
| `0xF70` | REG_CPU_OSC3_CTRL | CPU/OSC3 制御 |
| `0xF71` | REG_LCD_CTRL | LCD 制御 (初期値 0x8) |
| `0xF72` | REG_LCD_CONTRAST | LCD コントラスト |
| `0xF73` | REG_SVD_CTRL | SVD (電圧検出) 制御 |
| `0xF74` | REG_BUZZER_CTRL1 | ブザー周波数選択 |
| `0xF75` | REG_BUZZER_CTRL2 | ブザー制御 2 |
| `0xF76` | REG_CLK_WD_TIMER_CTRL | クロック/ウォッチドッグタイマー制御 |
| `0xF77` | REG_SW_TIMER_CTRL | ストップウォッチタイマー制御 |
| `0xF78` | REG_PROG_TIMER_CTRL | プログラマブルタイマー制御 |
| `0xF79` | REG_PROG_TIMER_CLK_SEL | プログラマブルタイマークロック選択 |
| `0xF7A` | REG_SERIAL_IF_CLK_SEL | シリアルインターフェースクロック選択 |
| `0xF7B` | REG_HIGH_IMPEDANCE_OUTPUT_CTRL | ハイインピーダンス出力制御 |
| `0xF7D` | REG_IO_CTRL | I/O 制御 |
| `0xF7E` | REG_IO_PULLUP_CFG | I/O プルアップ設定 |

### ブザー周波数 (REG_BUZZER_CTRL1)

| 値 | 周波数 |
|----|--------|
| 0 | 4096.0 Hz |
| 1 | 3276.8 Hz |
| 2 | 2730.7 Hz |
| 3 | 2340.6 Hz |
| 4 | 2048.0 Hz |
| 5 | 1638.4 Hz |
| 6 | 1365.3 Hz |
| 7 | 1170.3 Hz |

---

## 9. 割り込みシステム

### 割り込みスロット (優先度順)

| スロット | 定数 | ベクタ | 説明 |
|---------|------|--------|------|
| 0 (最高) | INT_PROG_TIMER_SLOT | `0x0C` | プログラマブルタイマー |
| 1 | INT_SERIAL_SLOT | `0x0A` | シリアルインターフェース |
| 2 | INT_K10_K13_SLOT | `0x08` | 入力 K10–K13 |
| 3 | INT_K00_K03_SLOT | `0x06` | 入力 K00–K03 |
| 4 | INT_STOPWATCH_SLOT | `0x04` | ストップウォッチタイマー |
| 5 (最低) | INT_CLOCK_TIMER_SLOT | `0x02` | クロックタイマー |

### 割り込み動作

割り込み発生時:
1. スタックに PC を退避 (PCP, PCSH, PCSL の順に3ニブル)
2. SP -= 3
3. I フラグをクリア (割り込み無効化)
4. `NP = TO_NP(NBP, 1)` → NP をページ1に設定
5. `PC = TO_PC(PCB, 1, vector)` → ページ1のベクタアドレスへジャンプ

割り込みハンドラは RET で戻る。I フラグは EI で再有効化する。

### 割り込み処理のスキップ条件

以下の命令の直後は割り込み処理がスキップされる:
- **PSET**: NP 設定直後に割り込みが入ると、NP が自動リセットされ PSET→JP のページジャンプが壊れるため
- **EI**: 割り込み有効化の直後に即座に割り込みが発生することを防止するため

---

## 10. タイマー

### クロックタイマー (自動更新)

各ビットは対応する周波数でトグル (XOR) される。

| 周波数 | レジスタ | 割り込み生成 |
|--------|---------|-------------|
| 128 Hz | REG_CLOCK_TIMER_DATA_1 bit0 | なし |
| 64 Hz | REG_CLOCK_TIMER_DATA_1 bit1 | なし |
| 32 Hz | REG_CLOCK_TIMER_DATA_1 bit2 | INT_CLOCK_TIMER_SLOT bit 0 (立ち下がりエッジ) |
| 16 Hz | REG_CLOCK_TIMER_DATA_1 bit3 | なし |
| 8 Hz | REG_CLOCK_TIMER_DATA_2 bit0 | INT_CLOCK_TIMER_SLOT bit 1 (立ち下がりエッジ) |
| 4 Hz | REG_CLOCK_TIMER_DATA_2 bit1 | なし |
| 2 Hz | REG_CLOCK_TIMER_DATA_2 bit2 | INT_CLOCK_TIMER_SLOT bit 2 (立ち下がりエッジ) |
| 1 Hz | REG_CLOCK_TIMER_DATA_2 bit3 | INT_CLOCK_TIMER_SLOT bit 3 (立ち下がりエッジ) |

> **注意**: 割り込みはビットが 1→0 に変化する（立ち下がりエッジ）時のみ生成される。
> 4Hz, 16Hz, 64Hz, 128Hz はデータレジスタの更新のみで、割り込みは生成しない。

### プログラマブルタイマー

- REG_PROG_TIMER_DATA_L/H: カウンタ値 (8bit、L+H の2ニブル)
- REG_PROG_TIMER_RELOAD_DATA_L/H: リロード値
- REG_PROG_TIMER_CLK_SEL: クロック源選択
- カウントが 0 になると INT_PROG_TIMER_SLOT 割り込みを生成

---

## 11. リセット後の初期状態

| レジスタ | 初期値 | 意味 |
|---------|--------|------|
| PC | `0x0100` | bank=0, page=1, step=0x00 |
| NP | `0x01` | bank=0, page=1 |
| A, B | `0x0` | (未定義、0 として扱う) |
| X, Y | `0x000` | |
| SP | `0x00` | |
| F | `0x0` | 全フラグ = 0 |
| REG_LCD_CTRL | `0x8` | LCD 制御初期値 |
| REG_K00_K03_INPUT_RELATION | `0xF` | Active HIGH |
| REG_R40_R43_BZ_OUTPUT_PORT | `0xF` | 出力ポート初期値 |
| メモリ全体 | `0x0` | ゼロクリア |

---

## 12. ROM 開発チートシート

### アドレス 0xExx へのポインタ設定手順

`LD X #imm` は XP を変更しないため、12bit アドレスの設定には3命令必要:

```
LD A, #0xE      ; A = 目的のページ番号 (例: 0xE)
LD XP, A        ; XP = 0xE
LD X, #0x00     ; X = 0xE00 (下位8bit を設定)
```

### 無限ループ

```
JP #0x00        ; → PC = 0x00 | (NP << 8) = 現在のページ先頭へ
```

自分自身へのループ (PC=0x0104 にある場合):
```
JP #0x04        ; → PC = 0x04 | (NP << 8) = 0x0104
```

### ページジャンプ

別ページへ飛ぶには先に PSET で NP を設定:
```
PSET #0x02      ; NP = 0x02 (ページ2)
JP #0x00        ; → PC = 0x0200
```

### ページ0のサブルーチン呼び出し

```
CALZ #0x50      ; ページ0のアドレス 0x50 を呼ぶ
                ; 戻り先はスタックに自動保存
```

### ディスプレイメモリへの書き込みパターン

**ポインタ経由 (連続書き込みに便利)**:
```
LD A, #0xE      ; XP = 0xE に設定
LD XP, A
LD X, #0x00     ; X = 0xE00
LDPX MX, #0xF   ; M[0xE00]=0xF, X→0xE01
LDPX MX, #0xF   ; M[0xE01]=0xF, X→0xE02
...
```

**直接アドレス (RAM 内のみ 0x00–0xFF)**:
```
LD A, #0x5
LD M(#0x05) A   ; M[0x05] = 0x5  ※0x0–0xFの範囲のみ
```

### 典型的なプログラム構造

```
; ページ1 (0x0100–0x01FF): エントリポイント・メインループ
0x0100:  DI              ; 割り込み無効
         ...             ; 初期化処理
0x0110:  EI              ; 割り込み有効
0x0111:  HALT            ; 次の割り込みまで待機 (省電力)
0x0112:  JP #0x11        ; ループ

; ページ1 のベクタ (0x0102, 0x0104, 0x0106, 0x0108, 0x010A, 0x010C)
0x0102:  CALL #0x?? 等   ; 各割り込みハンドラ
```

---

*このドキュメントは TamaLIB (tamalib/cpu.c, tamalib/hw.c) のソースコード解析に基づいています。*
