# 01 - Giriş

## 💡 Notlarım

### Neden Container Orchestration?

Bir e-ticaret sitesi düşünelim. Bu site tek parça değil; birden çok bileşenden oluşur:

- **Frontend** — kullanıcının gördüğü arayüz
- **Backend** — iş mantığının çalıştığı katman
- **Database** — verilerin saklandığı yer
- **Cache** — sık erişilen veriyi hızlandıran katman

Her bir bileşeni ayrı bir **container** olarak paketleyip (container image haline getirip) çalıştırabiliriz. Böylece her parça bağımsız, taşınabilir ve tekrarlanabilir olur.

**Peki sorun ne?** Bu container'lar çoğaldıkça (onlarca/yüzlerce, birden çok sunucuya dağılmış) elle yönetmek imkânsız hale gelir:
- Hangi container hangi sunucuda çalışacak?
- Biri çökerse kim yeniden başlatacak?
- Yük artınca nasıl ölçekleyeceğiz (kopya sayısını artırma)?
- Container'lar birbiriyle ve dış dünyayla nasıl haberleşecek?
- Güncelleme yaparken kesinti olmadan nasıl geçiş yapacağız?

İşte tüm bu işleri **otomatik** yapan sisteme **Container Orchestration** (konteyner orkestrasyonu) denir. Kubernetes de bu işi yapan en yaygın araçtır.

### Kubernetes Tarihçesi

- Google, yıllarca kendi iç sistemlerini **Borg** adlı platformla yönetti.
- Bu tecrübeyi açık kaynağa taşıyarak **2014**'te Kubernetes'i duyurdu.
- **2015**'te v1.0 çıktı ve proje **CNCF** (Cloud Native Computing Foundation) çatısına devredildi.
- İsim, Yunanca **"dümenci / kaptan"** anlamına gelir (gemiyi yöneten kişi).

### Kubernetes Nedir?

Kubernetes, container olarak paketlenmiş uygulamaları **otomatik olarak dağıtan, ölçekleyen ve yöneten** açık kaynaklı bir orchestration platformudur. Container'ların hangi sunucuda çalışacağına karar verir, çökenleri yeniden ayağa kaldırır, yükü dağıtır ve güncellemeleri kesintisiz yönetir.
