### Configuration (Yapılandırma)

*   **ConfigMaps**
    *   Ortam değişkenleri (environment variables) olarak kullanma
    *   Volume mount'ları ile dosya olarak kullanma
    *   Komut satırı argümanları için kullanma
    *   Değişmez (Immutable) ConfigMap'ler
*   **Secrets**
    *   Ortam değişkenleri (environment variables) olarak kullanma
    *   Volume mount'ları ile dosya olarak kullanma
    *   Docker imaj pull secret'ları (imagePullSecrets)
    *   Değişmez (Immutable) Secret'lar
*   **SecurityContexts (Güvenlik Bağlamları)**
    *   PodSecurityContext (Pod seviyesinde güvenlik ayarları)
    *   ContainerSecurityContext (Container seviyesinde güvenlik ayarları)
    *   Capabilities (yetkiler), `runAsUser`, `runAsGroup`, `fsGroup`, `SELinuxOptions`, `seccompProfile`, `readOnlyRootFilesystem` vb.
*   **ServiceAccounts**
    *   Pod'lara ServiceAccount atama (`automountServiceAccountToken`)
    *   Roles ve RoleBindings (Namespace kapsamlı yetkilendirme)
    *   (ClusterRoles ve ClusterRoleBindings hakkında temel bilgi de faydalı olabilir, ancak CKAD daha çok namespace'lere odaklanır)
*   **Resource Requirements (Kaynak Gereksinimleri)**
    *   İstekler (requests) ve Limitler (limits)
    *   CPU ve Bellek (Memory) için kaynak tanımlama
    *   Quality of Service (QoS) sınıfları (Guaranteed, Burstable, BestEffort)
*   **Commands and Arguments (Komutlar ve Argümanlar)**
    *   Docker imajının CMD ve ENTRYPOINT'ini override etme
*   **Taints and Tolerations (Lekeler ve Toleranslar)**
    *   Pod'ların belirli node'larda çalışmasını veya çalışmamasını sağlama
*   **Node Selectors (Düğüm Seçiciler)**
    *   Pod'ları etiketlere göre belirli node'lara atama
*   **Node Affinity (Düğüm Eğilimi)**
    *   Daha gelişmiş node seçimi kuralları
    *   `requiredDuringSchedulingIgnoredDuringExecution`
    *   `preferredDuringSchedulingIgnoredDuringExecution`
*   **Pod Affinity and Anti-Affinity (Pod Eğilimi ve Karşı Eğilimi)**
    *   Pod'ların birbirlerine göre (aynı veya farklı node'larda) konumlandırılması

### MultiContainer Pods (Çoklu Konteyner Pod'ları)

*   **Sidecar Pattern** (Yardımcı konteynerler: loglama, izleme, proxy vb.)
*   **Init Containers** (Pod başlamadan önce çalıştırılan hazırlık konteynerleri)
*   **Adapter and Ambassador Patterns** (Arayüz uyarlama ve dış servislerle iletişim için kullanılan desenler)

### Observability (Gözlemlenebilirlik)

*   **Readiness Probes** (Pod'un trafik kabul etmeye hazır olup olmadığını kontrol etme)
*   **Liveness Probes** (Pod'un sağlıklı çalışıp çalışmadığını kontrol etme, sağlıksızsa yeniden başlatma)
*   **Startup Probes** (Yavaş başlayan uygulamalar için başlangıç kontrolü)
*   **Container Logging (`kubectl logs`)**
    *   Çoklu konteyner pod'larında log okuma (`-c <container_name>`)
*   **Monitor and Debug Applications (Uygulamaları İzleme ve Hata Ayıklama)**
    *   `kubectl describe pod/deployment/service <name>` (Detaylı bilgi alma)
    *   `kubectl exec -it <pod_name> -- <command>` (Pod içinde komut çalıştırma)
    *   `kubectl port-forward pod/<pod_name> <local_port>:<pod_port>` (Lokalden pod'a erişim)
    *   `kubectl top pod/node` (Metrics Server kuruluysa kaynak kullanımını görme)
    *   Uygulama metriklerine temel bakış (örn: kaynak kullanımı, özel metrikler)

### POD Design (POD Tasarımı)

*   **Labels, Selectors and Annotations (Etiketler, Seçiciler ve Ek Açıklamalar)**
    *   Kaynakları organize etme ve seçme
*   **Deployments**
    *   Deployment Stratejileri: **Rolling Updates** (Kademeli Güncelleme), **Recreate** (Yeniden Oluşturma)
    *   Rollouts (Sürümler) ve Rollbacks (Geri Almalar) (`kubectl rollout status/history/undo`)
    *   Deployment yönetimi (`kubectl scale`, `kubectl set image`, `kubectl edit`)
    *   **Blue/Green, Canary** dağıtım stratejileri (Bunlar doğrudan Kubernetes nesneleri olmasa da, Deployment'lar ve Service'ler kullanılarak nasıl uygulanabileceğini anlamak önemlidir.)
*   **Jobs and CronJobs**
    *   Job tamamlama (`completions`) ve paralellik (`parallelism`) ayarları
    *   CronJob zamanlama (`schedule`) sözdizimi ve yönetimi (`startingDeadlineSeconds`, `concurrencyPolicy`)

### Services and Networking (Servisler ve Ağ İletişimi)

*   **Service Tipleri:**
    *   **ClusterIP** (Varsayılan, küme içi erişim)
    *   **NodePort** (Her node üzerinde statik bir port üzerinden erişim)
    *   **LoadBalancer** (Bulut sağlayıcısının yük dengeleyicisi ile dış erişim)
    *   **ExternalName** (Servis'i bir DNS adına yönlendirme)
    *   **Headless Services** (Yük dengeleme olmadan, doğrudan Pod IP'lerine erişim için, özellikle StatefulSet'ler ile kullanılır)
*   **Ingress Resource ve Ingress Controller (nginx, Traefik, ALB etc.)**
    *   Ingress kuralları (host tabanlı, path tabanlı yönlendirme)
    *   TLS sonlandırma (TLS termination) ve sertifika yönetimi
    *   Birden fazla Ingress Controller kullanımı (`ingressClassName`)
*   **Network Policies (Ağ Politikaları)**
    *   Pod'lar arası ağ trafiğini kontrol etme
    *   Giriş (Ingress) ve Çıkış (Egress) kuralları
    *   Pod seçimi (`podSelector`) ve namespace seçimi (`namespaceSelector`)
    *   Varsayılan (Default) deny/allow politikaları

### State Persistence (Durum Kalıcılığı)

*   **Volumes (Birimler)**
    *   Volume Tipleri: `emptyDir`, `hostPath` (geliştirme/test için dikkatli kullanılmalı), `configMap`, `secret`, `downwardAPI`, `projected`
*   **Persistent Volumes (PVs) (Kalıcı Birimler)**
    *   Yönetici tarafından sağlanan depolama kaynakları
*   **Persistent Volume Claims (PVCs) (Kalıcı Birim Talepleri)**
    *   Kullanıcıların depolama talepleri
*   **StorageClasses (Depolama Sınıfları)**
    *   Dinamik provizyonlama (dynamic provisioning) için
    *   Farklı depolama türleri ve politikaları tanımlama
*   **Access Modes (Erişim Modları)**
    *   `ReadWriteOnce` (RWO)
    *   `ReadOnlyMany` (ROX)
    *   `ReadWriteMany` (RWX)
    *   `ReadWriteOncePod` (RWOP) - Yeni sürümlerde

### Security (Güvenlik)

*   **RBAC (Rol Tabanlı Erişim Kontrolü)** - ServiceAccounts, Roles, RoleBindings (ve ClusterRoles/ClusterRoleBindings) ile Pod'ların API sunucusuna erişim yetkilerini yönetme.
*   **İmaj güvenliği (Image security)**
    *   Güvenilir kaynaklardan imaj kullanma
    *   Özel registry (private registry) kullanımı ve `imagePullSecrets`
    *   İmajların zafiyet taraması (kavramsal olarak bilinmesi)
*   **API Grupları ve Kaynaklara Erişim** - Bir ServiceAccount'ın veya uygulamanın hangi API kaynaklarına (pods, services, configmaps vb.) hangi operasyonları (get, list, watch, create, update, patch, delete) yapabileceğini anlama.
*   **Minimum Yetki Prensibi (Principle of Least Privilege)** - Pod'lara ve ServiceAccount'lara sadece ihtiyaç duydukları minimum yetkileri verme.
*   **Secrets yönetimi** - Hassas verilerin güvenli bir şekilde saklanması ve Pod'lara iletilmesi.

### Helm Fundamentals (Helm Temelleri)

*   **Chart'lar (Charts)**
    *   Chart yapısı (`Chart.yaml`, `values.yaml`, `templates/` dizini, `NOTES.txt`)
    *   Bağımlılıklar (`dependencies` - `requirements.yaml` eski versiyonlarda)
*   **Release'ler (Sürümler)** - Bir Chart'ın Kubernetes kümesine kurulmuş bir örneği.
*   **Repository'ler (Depolar)** - Chart'ların saklandığı ve paylaşıldığı yerler.
*   **Template'ler (Şablonlar)**
    *   Go templating dili temelleri
    *   Yerleşik nesneler: `.Values`, `.Release`, `.Chart`, `.Capabilities`
    *   Fonksiyonlar ve pipeline'lar
    *   `_helpers.tpl` kullanımı
*   **Temel Helm Komutları**
    *   `helm create`
    *   `helm install`
    *   `helm upgrade`
    *   `helm rollback`
    *   `helm list`
    *   `helm status`
    *   `helm uninstall`
    *   `helm template` (Render edilmiş manifestleri görmek için)
    *   `helm lint` (Chart'ı kontrol etmek için)
    *   `helm package`
    *   `helm repo add/update/list/remove`
*   **Değer (Values) Yönetimi**
    *   `values.yaml` dosyası
    *   `--set` ve `--set-string` ile komut satırından değer geçme
    *   `--values` (`-f`) ile ek değer dosyaları kullanma
