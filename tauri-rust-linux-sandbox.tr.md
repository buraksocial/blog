---
title: "Tauri ve Rust ile Linux'ta Sınırları Aşmak: Native OS Entegrasyonu ve Asenkron Yetki Yönetimi"
date: "2026-08-25"
description: "Modern bir masaüstü aracı veya paket yöneticisi geliştirirken, Wayland ve Polkit mimarilerinde güvenlikten (sandbox) ödün vermeden root yetkileri asenkron olarak nasıl yönetilir?"
tags: "linux, tauri, rust, polkit, wayland, security, archlinux, development"
banner: ""
---

Web teknolojileriyle performanslı masaüstü uygulamaları geliştirmek söz konusu olduğunda Tauri, düşük bellek tüketimi ve Rust tabanlı güvenli backend mimarisiyle öne çıkıyor. Klasik bir To-Do uygulaması veya API istemcisi yapıyorsanız her şey harikadır. Ancak iş işletim sisteminin sinir sistemine inmeye geldiğinde —örneğin Arch Linux için pacman veya paru ile haberleşen bir masaüstü paket yöneticisi inşa ederken— oyunun kuralları tamamen değişir.

Rust, Tauri ve React kullanarak **PackageHub** gibi bir masaüstü paket yöneticisi geliştirirken, modern Linux sistemlerinde (özellikle Wayland üzerinde) karşılaştığım temel mimari zorlukları ve bunları nasıl çözdüğümü adım adım anlatacağım.

## Sorun: Wayland, GUI Uygulamaları ve Root Yetkisi

Eski X11 günlerinde bir uygulamanın root yetkisine ihtiyacı varsa, onu `gksudo` veya `pkexec` ile çalıştırırdınız ve tüm grafik arayüzlü uygulama root olarak çalışırdı. Ancak bu büyük bir güvenlik zafiyetiydi.

Wayland gibi modern görüntü sunucuları, **GUI (grafik arayüzlü) uygulamaların root olarak çalışmasını kesinlikle yasaklar**. Root olarak çalışan bir grafik uygulaması, diğer tüm uygulamaların belleğine ve tuş vuruşlarına (keylogger) erişebilir. Bu yüzden bir Tauri uygulamasını `sudo` ile çalıştırmaya kalktığınızda, Wayland pencereyi açmayı basitçe reddeder veya çöker.

Peki, root yetkisi gerektiren bir paket kurma komutunu (`pacman -S <paket>`) nasıl çalıştıracağız, terminal çıktısını anlık olarak React arayüzüne nasıl aktaracağız ve tüm bu süreci nasıl güvenli tutacağız?

## Çözüm Konsepti: Yetki Ayrılığı (Privilege Separation) ve IPC

Bunu güvenli bir şekilde yapmanın tek yolu **Yetki Ayrılığı (Privilege Separation)** prensibidir.

1. **Önyüz (React) + Arkayüz (Tauri/Rust):** Standart, yetkisiz bir kullanıcı olarak çalışır. Arayüzü, API isteklerini ve zararsız işlemleri (paket arama gibi) halleder.
2. **Worker Süreci (Polkit):** Sadece *gerektiğinde* root olarak çalıştırılan, arayüzü olmayan ayrı bir Rust derlemesi (veya komut dosyası).
3. **İletişim (IPC):** Yetkisiz Tauri uygulaması, yetkili Worker süreci ile standart girdi/çıktı (stdio) veya Unix Domain Sockets üzerinden haberleşir.

### 1. Polkit (PolicyKit) Entegrasyonu

Yetki almak için `sudo` yerine `pkexec` kullanmalıyız. `pkexec`, modern Linux sistemlerinde yetki yönetimi için kullanılan standart araçtır. Kullanıcıya yerel, şık bir parola ekranı çıkartır.

```rust
// tauri/src/commands.rs
use std::process::{Command, Stdio};
use tauri::Window;

#[tauri::command]
pub async fn install_package(window: Window, package_name: String) -> Result<(), String> {
    // Komutumuzu pkexec ile sarmalıyoruz
    let mut child = Command::new("pkexec")
        .arg("pacman")
        .arg("-S")
        .arg("--noconfirm")
        .arg(&package_name)
        .stdout(Stdio::piped())
        .spawn()
        .map_err(|e| e.to_string())?;
        
    // ... artık stdout akışını okuyabiliriz
    Ok(())
}
```

### 2. React'a Asenkron Aktarım (Zorluk Burada)

Yukarıdaki komut çalışır, ancak paket kurulumları zaman alır. Kullanıcının bir ilerleme çubuğu veya terminal çıktısı görebilmesi için `pacman`'in çıktısını satır satır React önyüzüne aktarmamız gerekir.

Bunu yapmak için `child.stdout`'u asenkron olarak okumalıyız. Ancak, `std::process::Command` senkron çalışır ve thread'i (iş parçacığını) kilitler. Bir Tauri uygulamasında ana thread'i kilitlemek, arayüzün donması demektir!

Bu noktada imdadımıza `tokio::process::Command` yetişiyor.

```rust
use tokio::process::Command;
use tokio::io::{AsyncBufReadExt, BufReader};
use std::process::Stdio;
use tauri::Window;

#[tauri::command]
pub async fn install_package(window: Window, package_name: String) -> Result<(), String> {
    let mut child = Command::new("pkexec")
        .arg("pacman")
        .arg("-S")
        .arg("--noconfirm")
        .arg(&package_name)
        .stdout(Stdio::piped())
        .spawn()
        .map_err(|e| e.to_string())?;

    let stdout = child.stdout.take().expect("Stdout açılamadı");
    let mut reader = BufReader::new(stdout).lines();

    // Akışı bloklanmadan okumak için yeni bir asenkron görev (task) başlatıyoruz
    tokio::spawn(async move {
        while let Ok(Some(line)) = reader.next_line().await {
            // Satırı anında React arayüzüne (frontend) fırlatıyoruz
            window.emit("package-install-log", Some(line)).unwrap();
        }
    });

    Ok(())
}
```

### 3. React Tarafında Karşılama

Artık frontend tarafında, tek yapmamız gereken bu olayı (event) dinlemek:

```typescript
import { listen } from '@tauri-apps/api/event';
import { useEffect, useState } from 'react';

function Terminal() {
    const [logs, setLogs] = useState<string[]>([]);

    useEffect(() => {
        const unlisten = listen('package-install-log', (event) => {
            setLogs((prev) => [...prev, event.payload as string]);
        });

        return () => {
            unlisten.then(f => f());
        };
    }, []);

    return (
        <div className="bg-black text-green-400 p-4 font-mono">
            {logs.map((log, i) => <div key={i}>{log}</div>)}
        </div>
    );
}
```

## Sonuç

Yetki yükseltme için `pkexec`'i ve asenkron akışlar için `tokio`'yu kullanarak güvenli, Wayland uyumlu bir paket yöneticisi arayüzü inşa etmiş olduk. Bu model (asenkron IPC ile eşleştirilmiş yetki ayrılığı), modern Linux masaüstü geliştirme dünyasının altın standardıdır.

Tauri, Rust'ın sistem seviyesindeki güvenliği ile React'ın muhteşem UX yetenekleri arasında mükemmel bir köprü sağlıyor.
