---
title: "Tauri ve Rust ile Linux'ta Sınırları Aşmak: Native OS Entegrasyonu ve Asenkron Yetki Yönetimi"
image: "https://i.ibb.co/RNHgsbD/v5-open-source.png"
tags: "linux, rust, tauri, web, geliştirme"
date: "2026-08-25"
---

Modern bir masaüstü aracı veya paket yöneticisi geliştirirken, Wayland ve Polkit mimarilerinde güvenlikten (sandbox) ödün vermeden root yetkileri asenkron olarak nasıl yönetilir?

Web teknolojileriyle performanslı masaüstü uygulamaları geliştirmek söz konusu olduğunda Tauri, düşük bellek tüketimi ve Rust tabanlı güvenli backend mimarisiyle öne çıkıyor. Klasik bir To-Do uygulaması veya API istemcisi yapıyorsanız her şey harikadır. Ancak iş işletim sisteminin sinir sistemine inmeye geldiğinde —örneğin Arch Linux için `pacman` veya `paru` ile haberleşen bir paket yöneticisi inşa ederken— oyunun kuralları tamamen değişir.

Rust, Tauri ve React kullanarak PackageHub gibi bir masaüstü paket yöneticisi geliştirirken karşılaşılan en büyük mimari problem şudur: Tauri'nin güvenli sandbox yapısını kırmadan, Wayland'in katı güvenlik kurallarına takılmadan ve UI thread'ini bloklamadan root yetkisi gerektiren komutları nasıl çalıştırırsınız?

Bu makalede, klasik `Command::new("sudo")` tuzaklarına düşmeden, Linux ortamında doğru ve modern yetki yönetimi mimarisinin nasıl kurgulanacağını inceleyeceğiz.

## Neden sudo Kullanmamalıyız? (Wayland'in Kuralları)

Linux dünyasında X11'den Wayland'e geçiş, güvenlik anlamında büyük bir paradigma değişimidir. Wayland, grafiksel bir uygulamanın doğrudan root olarak çalıştırılmasını (tasarımı gereği) engeller.

Eğer bir paket yöneticisi yapıyorsanız ve arka planda `pacman -Syu` komutunu çalıştırmanız gerekiyorsa, aklınıza ilk gelen şey uygulamanın arka ucundan terminale bir `sudo` isteği yollamak olabilir. Ancak bu yaklaşım üç ölümcül hata barındırır:

- **İnteraktiflik**: `sudo` şifre beklediğinde bunu Tauri'nin görünmez backend'inden kullanıcıya GUI üzerinden yansıtmak bir kabustur.
- **UI Bloklanması**: Senkron bir süreç başlatırsanız, paketler indirilirken frontend tamamen donar.
- **Güvenlik İhlali**: Tüm uygulamaya root yetkisi vermek, en ufak bir XSS açığında tüm sistemi tehlikeye atar.

Çözüm, **Yetki Ayrılığı (Privilege Separation)** ve **Asenkron İletişim (Asynchronous IPC)** mimarisini birleştirmektir.

## Doğru Mimari: Polkit ve Tokio ile Asenkron Orkestrasyon

Modern Linux sistemlerinde bir GUI uygulamasının yetki yükseltme (privilege escalation) standardı `sudo` değil, Polkit (PolicyKit) üzerinden çalışan `pkexec`'tir. `pkexec`, root işlemi için sistemin kendi native ve güvenli şifre sorma arayüzünü (authentication agent) tetikler.

İşte Tauri backend'inde bu işleyişi kurmanın asenkron (Non-Blocking) yolu:

### 1. Tauri Command'ını Asenkron Hale Getirmek

Paket yükleme veya sistem güncelleme işlemleri dakikalar sürebilir. Bu süreci Rust'ın asenkron runtime'ı olan Tokio ile yönetmeliyiz.

```rust
use tauri::Window;
use tokio::process::Command;
use std::process::Stdio;
use tokio::io::{AsyncBufReadExt, BufReader};

#[tauri::command]
async fn execute_pacman_install(window: Window, package_name: String) -> Result<(), String> {
    // Polkit kullanarak yetki istiyoruz ve pacman komutunu çalıştırıyoruz
    let mut child = Command::new("pkexec")
        .arg("pacman")
        .arg("-S")
        .arg("--noconfirm")
        .arg(&package_name)
        .stdout(Stdio::piped())
        .stderr(Stdio::piped())
        .spawn()
        .map_err(|e| format!("İşlem başlatılamadı: {}", e))?;

    let stdout = child.stdout.take().expect("Stdout yakalanamadı");
    let mut reader = BufReader::new(stdout).lines();

    // Çıktıyı asenkron olarak okuyup Frontend'e (React/Next.js) iletiyoruz
    tokio::spawn(async move {
        while let Ok(Some(line)) = reader.next_line().await {
            // Tauri event sistemi ile terminal loglarını UI'a stream ediyoruz
            window.emit("pacman-log", Some(line)).unwrap();
        }
    });

    let status = child.wait().await.map_err(|e| e.to_string())?;
    
    if status.success() {
        Ok(())
    } else {
        Err("Paket yükleme başarısız oldu.".to_string())
    }
}
```

### 2. Frontend'de Veriyi Gerçek Zamanlı Yakalamak (React)

Tauri'nin event sistemi sayesinde, Rust tarafındaki `tokio::spawn` içinden gönderilen her bir log satırını, frontend tarafında sanki bir terminal ekranıymış gibi anlık olarak gösterebiliriz.

```javascript
import { listen } from '@tauri-apps/api/event';
import { useEffect, useState } from 'react';

function InstallerTerminal() {
  const [logs, setLogs] = useState([]);

  useEffect(() => {
    const unlisten = listen('pacman-log', (event) => {
      setLogs((prevLogs) => [...prevLogs, event.payload]);
    });

    return () => {
      unlisten.then(f => f());
    };
  }, []);

  return (
    <div className="bg-gray-900 text-green-400 p-4 font-mono">
      {logs.map((log, i) => <div key={i}>{log}</div>)}
    </div>
  );
}
```

## Neler Başardık?

Bu mimariyi uygulayarak bir masaüstü yazılımının sahip olması gereken en profesyonel standartları yakalamış olduk:

- **Wayland Uyumluluğu**: UI katmanımız standart kullanıcı (user space) yetkileriyle güvenli bir şekilde çalışırken, sadece spesifik paket yükleme komutu (child process) `pkexec` ile root yetkisine yükseldi.
- **Gerçek Zamanlı Streaming**: Rust tarafında `tokio::process` kullanarak I/O işlemlerini non-blocking hale getirdik. Kullanıcı, terminaldeki gibi yükleme yüzdelerini React arayüzünde anlık olarak görebiliyor.
- **Sıfır Sandbox İhlali**: Tauri'nin güvenlik modeli delinmedi. Root yetkisi uygulamaya değil, sadece çalıştırılan spesifik komuta verildi.

## Sonuç

Rust ve Tauri ile sistem entegrasyonu araçları geliştirmek, sadece bir frontend çizmekten çok daha fazlasıdır; işletim sisteminin alt katmanlarına, IPC (Inter-Process Communication) mekanizmalarına ve asenkron I/O yönetimine hakim olmayı gerektirir. Paket yöneticileri, sistem monitörleri veya OS optimizasyon araçları geliştirirken bu mimari, projenizi amatör bir betikten çıkarıp enterprise seviyesinde bir uygulamaya dönüştürür.

Eğer siz de benzer mimariler üzerine çalışıyorsanız veya masaüstünde web teknolojilerinin sınırlarını zorlamayı seviyorsanız, deneyimlerinizi paylaşmaktan çekinmeyin!
