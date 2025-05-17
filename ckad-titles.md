### Configuration (Yapılandırma)
*   **ConfigMaps**
*   **Secrets**
*   **SecurityContexts (Güvenlik Bağlamları)**
*   **Resource Requirements (Kaynak Gereksinimleri)**
*   **Commands and Arguments (Komutlar ve Argümanlar)**
*   **Taints and Tolerations (Lekeler ve Toleranslar)**
*   **Node Selectors (Düğüm Seçiciler)**
*   **Node Affinity (Düğüm Eğilimi)**
*   **Pod Affinity and Anti-Affinity (Pod Eğilimi ve Karşı Eğilimi)**

### MultiContainer Pods (Çoklu Konteyner Pod'ları)
*   **Sidecar Pattern**
*   **Init Containers**
*   **Adapter and Ambassador Patterns**

### Observability (Gözlemlenebilirlik)
*   **Readiness, Liveness, Startup Probes**
*   **Container Logging (`kubectl logs`)**
*   **Monitor and Debug Applications**

### POD Design (POD Tasarımı)
*   **Labels, Selectors and Annotations (Etiketler, Seçiciler ve Ek Açıklamalar)**
*   **Deployments**
*   **Jobs and CronJobs**
*   **PodDisruptionBudgets (PDBs)**
*   **DaemonSets**
*   **StatefulSets**

### Services and Networking (Servisler ve Ağ İletişimi)
*   **Service Tipleri**
*   **Ingress Resource ve Ingress Controller**
*   **Network Policies (Ağ Politikaları) (Ingress/Egress Rules)**

### State Persistence (Durum Kalıcılığı)
*   **Volumes (Birimler)**
*   **Persistent Volumes (PVs) ve Persistent Volume Claims (PVCs)**
*   **StorageClasses (Depolama Sınıfları)**
*   **Access Modes (Erişim Modları)**
*   **StorageClass, PV, PVC ve Pod'a Ekleme Entegrasyonu**

### Security (Güvenlik)
*   **Authentication: Kubeconfig basics**
*   **İmaj güvenliği (Image security)**
*   **RBAC: ServiceAccounts, Roles, RoleBindings, ClusterRoles, ClusterRoleBindings**
*   **Admission Controllers**
*   **API Grupları ve Kaynaklara Erişim, API Versions/Deprecations**
*   **Custom Resource Definitions (CRDs)**
*   **Minimum Yetki Prensibi (Principle of Least Privilege)**
*   **Secrets yönetimi**

### ResourceQuotas ve LimitRanges
*   **ResourceQuotas**
*   **LimitRanges**

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
 

