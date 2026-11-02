# beagleboardManifest

BeagleBone Black (BBB) üzerinde **Automotive Grade Linux (AGL) Unagi** çalıştırmak için kaynak ağacını kuran [repo](https://gerrit.googlesource.com/git-repo) manifest deposu.

- Yocto ve AGL katmanları (poky, meta-openembedded, meta-agl, meta-ti ...) belirli commit'lere **sabitlenmiştir**.
- BBB katmanı **meta-BBB** `main` dalını izler: https://github.com/draxh-es/meta-BBB

Kartın kendisi, derleme, SD karta yazma ve boot adımları için meta-BBB README'sine bakın.

---

## Neden ayrı bir katman?

AGL, Unagi sürümünde BeagleBone şablonunu kaldırdı (meta-agl `91ef89dc`, "Remove beaglebone templates"). O tarihte scarthgap için BBB BSP'si yoktu. Bugün meta-ti'de `meta-beagle/conf/machine/beaglebone.conf` mevcut (kernel `linux-bb.org` 6.12, `u-boot-bb.org` 2025.x). AGL'nin geri kalan BBB parçaları da hâlâ yerinde:

| Parça | Yer |
|---|---|
| AGL makine ayarları | `meta-agl/meta-agl-bsp/conf/include/agl_beaglebone.inc` |
| AGL kernel config | `meta-agl/meta-agl-bsp/meta-ti-bsp/recipes-kernel/linux/linux-bb.org_%.bbappend` |
| SD kart düzeni | `meta-agl/meta-agl-bsp/wic/sdimage-bootpart-uuid.wks` |

Eksik olan tek şey `aglsetup.sh -m beaglebone` şablonuydu. `aglsetup.sh`, ağacın kökündeki her `meta-*/templates/machine/<makine>` dizinini tarıyor. Bu yüzden şablon AGL'yi forklamadan meta-BBB içine kondu.

---

## Bu depoda neler var

| Dosya | Açıklama |
|---|---|
| `default.xml` | Giriş noktası. `agl-unagi-pinned.xml` dosyasını dahil eder ve meta-BBB'yi ağaca ekler. |
| `agl-unagi-pinned.xml` | AGL Unagi (21.0.4) ağacındaki tüm projeler, her biri commit SHA'sına sabit. `repo manifest -r` ile üretilir, **elle düzenlenmez**. |
| `README.md` | Bu dosya. |

`default.xml` içindeki `<remote name="beagle" fetch="."/>` satırı "bu manifest deposuyla aynı GitHub hesabı" anlamına gelir. `repo init` HTTPS ile yapıldıysa katman da HTTPS ile, SSH ile yapıldıysa SSH ile indirilir.

---

## Gereksinimler

| Kaynak | Öneri |
|---|---|
| İşletim sistemi | Ubuntu 22.04 (test edilen) |
| Disk | En az 150 GB boş alan |
| RAM | 16 GB ve üzeri |

```
sudo apt install build-essential chrpath cpio debianutils diffstat file \
  gawk gcc git iputils-ping libacl1 liblz4-tool locales python3 \
  python3-git python3-jinja2 python3-pexpect python3-pip python3-subunit \
  socat texinfo unzip wget xz-utils zstd bmap-tools picocom
sudo locale-gen en_US.UTF-8
```

repo aracı:

```
mkdir -p ~/bin
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
export PATH=~/bin:$PATH
```

---

## Hızlı başlangıç

```
mkdir -p ~/agl-bbb && cd ~/agl-bbb
repo init -u https://github.com/draxh-es/beagleboardManifest.git -b main
repo sync -j8

source meta-agl/scripts/aglsetup.sh -m beaglebone -b build-bbb agl-devel
bitbake agl-image-minimal
```

- `-m beaglebone`: meta-BBB'deki makine şablonu. `aglsetup.sh -h` çıktısında `[meta-BBB]` başlığı altında görünür.
- `agl-devel`: Şifresiz root ve hata ayıklama araçları. Üretim imajında kullanılmaz.

Yeni bir terminalde ortamı yüklemek için:

```
cd ~/agl-bbb
source build-bbb/agl-init-build-env
```

Sonraki adımlar (SD karta yazma, boot, seri konsol): meta-BBB README.

---

## Kaynak ağacının yapısı

```
~/agl-bbb/
├── .repo/            repo'nun kendi verileri
├── bsp/              Donanım katmanları (meta-ti, meta-arm ...)
├── external/         Yocto ve topluluk katmanları (poky, meta-openembedded ...)
├── meta-agl*/        AGL katmanları ve aglsetup.sh
├── meta-BBB/         BBB katmanı (bizim)
└── build-bbb/        Build dizini (hiçbir depoya ait değil)
```

---

## Sürüm politikası

| Bileşen | Nasıl izlenir | Neden |
|---|---|---|
| AGL, Yocto ve diğer açık kaynak katmanlar | Commit SHA'sına sabit | Dışarıdaki bir değişiklik derlemeyi habersizce bozmasın. AGL'nin CI'ı bu kartı artık test etmiyor. |
| meta-BBB | `main` dalı | Geliştirme sürerken en güncel hali alınsın |

Kartta doğrulanmış bir ağacın tam fotoğrafını almak için:

```
cd ~/agl-bbb
repo manifest -r -o releases/bbb-v0.1.0.xml   # bu depoya kopyalayıp commit'leyin, etiketleyin
```

O sürümü birebir kurmak için:

```
repo init -u https://github.com/draxh-es/beagleboardManifest.git -b main -m releases/bbb-v0.1.0.xml
```

---

## AGL sürümünü güncelleme

`agl-unagi-pinned.xml` elle düzenlenmez:

1. Yeni AGL ağacını kurun: `repo init -u https://gerrit.automotivelinux.org/gerrit/AGL/AGL-repo -b <dal-veya-etiket>` ve `repo sync`.
2. meta-BBB'yi ağacın köküne klonlayın, `agl-image-minimal` derleyip kartta test edin.
3. Sorun yoksa: `repo manifest -r -o <bu-depo>/agl-unagi-pinned.xml` ve 2. satıra bilgi notunu geri ekleyin.

AGL'nin farklı bir ana sürümüne geçiliyorsa `meta-BBB/conf/layer.conf` içindeki `LAYERSERIES_COMPAT_agl-bbb` değerini yeni Yocto sürümüne göre güncelleyin. AGL'deki `agl_beaglebone.inc`, `linux-bb.org_%.bbappend` ve meta-ti'deki `beaglebone.conf` dosyalarının hâlâ var olup olmadığını kontrol edin.

---

## Sorun giderme

| Belirti | Sebep ve çözüm |
|---|---|
| `aglsetup.sh -m beaglebone` "unknown machine" | meta-BBB ağacın kökünde değil. `repo sync` yapın ya da `ls meta-BBB/templates/machine` ile kontrol edin. |
| Katman eklenirken meta-python bağımlılık hatası | `conf/bblayers.conf`'a `AGL_META_PYTHON = "${METADIR}/external/meta-openembedded/meta-python"` ekleyin. |
| `Layer agl-bbb depends on layer meta-beagle` | `bblayers.conf`'ta `bsp/meta-ti/meta-beagle` yok. Build dizinini `aglsetup.sh` ile yeniden oluşturun. |
| Görevler `SIGKILL` ile ölüyor | RAM yetmiyor. `BB_NUMBER_THREADS` ve `PARALLEL_MAKE` değerlerini düşürün. |
| `No space left on device` | `local.conf`'a `INHERIT += "rm_work"` ekleyin. |
