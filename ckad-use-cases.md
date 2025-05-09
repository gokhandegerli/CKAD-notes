### Genel Sinav Stratejileri
*   Namespace/Context: Her zaman doğru namespace'de olduğunuzdan emin olun.
*   Imperative Komutlar: kubectl create/run/expose/set/scale/label/annotate/rollout gibi komutları önceliklendirin.
*   YAML (--dry-run=client -o yaml ve kubectl get ... -o yaml): Imperative komutların yetmediği yerde hızlıca YAML oluşturup düzenleyin.
*   kubectl explain: YAML alanları için vazgeçilmez yardımcınız. kubeclt <command> -h de etkilidir. 

### Configuration (Yapılandırma)

*   **ConfigMaps**
    *   **Çalışma Mantığı Notu:** ConfigMap'ler ve Secret'lar Pod'a mount edildiğinde, eğer ConfigMap/Secret güncellenirse, mount edilen dosyalar da (bir süre sonra, kubelet senkronizasyon periyoduna bağlı olarak) güncellenir. Ancak, ortam değişkeni olarak kullanılanlar Pod yeniden başlatılmadıkça güncellenmez. `immutable: true` ile bu davranış değiştirilebilir ve ConfigMap/Secret'ın içeriğinin değiştirilmesi engellenir, bu da yanlışlıkla yapılan değişikliklere karşı koruma sağlar ve küme performansını artırabilir (kubelet'in değişiklikleri izlemesi gerekmez).
    *   **Use Case 1: Ortam Değişkeni Olarak Kullanma (Tüm Anahtarlar)**
        *   **Senaryo:** `app-settings` adında bir ConfigMap oluşturun (`api_url=http://service-a`, `feature_flag=true` verileriyle). Bu ConfigMap'teki tüm anahtar-değer çiftlerini `my-app-pod` adlı bir Pod'un `app-container` adlı container'ına ortam değişkeni olarak atayın.
        *   **Çözüm Yaklaşımı:** Önce ConfigMap oluşturulur, sonra Pod manifestosunda `envFrom` ve `configMapRef` kullanılır.
        *   **Efektif Çözüm:**
            1.  `kubectl create configmap app-settings --from-literal=api_url=http://service-a --from-literal=feature_flag=true -n <namespace>`
            2.  Pod YAML'ı (`pod.yaml`):
                ```yaml
                apiVersion: v1
                kind: Pod
                metadata:
                  name: my-app-pod
                spec:
                  containers:
                  - name: app-container
                    image: busybox
                    command: ["sleep", "3600"]
                    envFrom:
                    - configMapRef:
                        name: app-settings
                ```
            3.  `kubectl apply -f pod.yaml -n <namespace>`
            4.  Kontrol: `kubectl exec my-app-pod -c app-container -n <namespace> -- env | grep -E "API_URL|FEATURE_FLAG"`
    *   **Use Case 2: Ortam Değişkeni Olarak Kullanma (Belirli Anahtar)**
        *   **Senaryo:** `app-settings` ConfigMap'indeki `api_url` anahtarının değerini `SERVICE_ENDPOINT` adlı bir ortam değişkeni olarak `my-app-pod`'a atayın.
        *   **Çözüm Yaklaşımı:** Pod manifestosunda `env` altında `valueFrom` ve `configMapKeyRef` kullanılır.
        *   **Efektif Çözüm:** Pod YAML'ı düzenlenir:
            ```yaml
            # ... (yukarıdaki Pod YAML'ının devamı)
            env:
            - name: SERVICE_ENDPOINT
              valueFrom:
                configMapKeyRef:
                  name: app-settings
                  key: api_url
            # ...
            ```
    *   **Use Case 3: Volume Mount Olarak Kullanma (Tüm Anahtarlar)**
        *   **Senaryo:** `app-settings` ConfigMap'ini `my-app-pod`'a `/etc/appconfig` yoluna bir volume olarak mount edin. ConfigMap'teki her anahtar bu dizin altında bir dosya olarak görünür.
        *   **Çözüm Yaklaşımı:** Pod manifestosunda `volumes` altında `configMap` ve `containers.volumeMounts` tanımlanır.
        *   **Efektif Çözüm:** Pod YAML'ı düzenlenir:
            ```yaml
            # ...
            spec:
              containers:
              - name: app-container
                # ...
                volumeMounts:
                - name: config-volume
                  mountPath: /etc/appconfig
              volumes:
              - name: config-volume
                configMap:
                  name: app-settings
            # ...
            ```
            Kontrol: `kubectl exec my-app-pod -c app-container -n <namespace> -- ls /etc/appconfig` (api_url ve feature_flag dosyalarını görmelisiniz)
    *   **Use Case 4: Volume Mount Olarak Kullanma (Belirli Anahtarlar / Dosya Yolu Değiştirme)**
        *   **Senaryo:** `nginx-custom-config` ConfigMap'indeki `custom.conf` anahtarını `web-server-pod`'a `/etc/nginx/conf.d/my_custom_nginx.conf` olarak mount edin.
        *   **Çözüm Yaklaşımı:** Pod manifestosunda `volumes` altında `configMap.items` ve `containers.volumeMounts` (gerekirse `subPath` ile) kullanılır.
        *   **Efektif Çözüm:** (Önce `nginx-custom-config` ConfigMap'inin `custom.conf` anahtarıyla oluşturulduğunu varsayalım: `kubectl create configmap nginx-custom-config --from-file=custom.conf=/path/to/your/custom.conf -n <namespace>`)
            Pod YAML'ı:
            ```yaml
            spec:
              containers:
              - name: nginx-container
                image: nginx
                volumeMounts:
                - name: nginx-custom-conf-volume
                  mountPath: /etc/nginx/conf.d/my_custom_nginx.conf # Mount edilecek dosyanın tam yolu
                  subPath: custom.conf # Volume içinde bu anahtarın içeriği bu dosya adıyla mount edilir
              volumes:
              - name: nginx-custom-conf-volume
                configMap:
                  name: nginx-custom-config
                  items:
                  - key: custom.conf
                    path: custom.conf # Volume içinde bu isimle dosya oluşturulur (subPath ile eşleşir)
            ```
    *   **Use Case 5: Değişmez (Immutable) ConfigMap Oluşturma**
        *   **Senaryo:** `stable-app-ver` adında, `version=v1.0.5` verisiyle değişmez bir ConfigMap oluşturun.
        *   **Çözüm Yaklaşımı:** ConfigMap manifestosunda `immutable: true` ayarlanır.
        *   **Efektif Çözüm:**
            ```yaml
            apiVersion: v1
            kind: ConfigMap
            metadata:
              name: stable-app-ver
            data:
              version: "v1.0.5"
            immutable: true
            ```
            `kubectl apply -f configmap.yaml -n <namespace>`
*   **Secrets**
    *   **Çalışma Mantığı Notu:** Secret'lar Kubernetes içinde base64 encode edilmiş olarak saklanır, bu bir şifreleme yöntemi değildir, sadece verinin okunabilir karakter setinde olmasını sağlar. Hassas verilerin korunması için etcd seviyesinde şifreleme, RBAC ile erişim kontrolü ve uygun Secret yönetim araçları (örn: HashiCorp Vault, Sealed Secrets) kullanılmalıdır. CKAD kapsamında temel Secret oluşturma ve kullanma becerisi önemlidir.
    *   **Use Case 1: Ortam Değişkeni Olarak Kullanma**
        *   **Senaryo:** `api-keys` adında bir Secret oluşturun (`api_user=service_account`, `api_token=supersecrettoken123`). Bu Secret'taki `api_token` anahtarını `API_AUTH_TOKEN` ortam değişkeni olarak bir Pod'a atayın.
        *   **Çözüm Yaklaşımı:** Önce Secret oluşturulur, sonra Pod YAML'ında `env.valueFrom.secretKeyRef` kullanılır.
        *   **Efektif Çözüm:**
            1.  `kubectl create secret generic api-keys --from-literal=api_user=service_account --from-literal=api_token=supersecrettoken123 -n <namespace>`
            2.  Pod YAML'ında:
                ```yaml
                env:
                - name: API_AUTH_TOKEN
                  valueFrom:
                    secretKeyRef:
                      name: api-keys
                      key: api_token
                ```
    *   **Use Case 2: Volume Mount Olarak Kullanma**
        *   **Senaryo:** `tls-certs` Secret'ını (içinde `tls.crt` ve `tls.key` anahtarları olduğunu varsayalım) bir Pod'a `/etc/ssl/private` yoluna mount edin.
        *   **Çözüm Yaklaşımı:** Pod manifestosunda `volumes` altında `secret` ve `containers.volumeMounts` tanımlanır.
        *   **Efektif Çözüm:** (Önce `tls-certs` Secret'ının oluşturulduğunu varsayalım: `kubectl create secret tls tls-certs --cert=/path/to/tls.crt --key=/path/to/tls.key -n <namespace>`)
            Pod YAML'ında:
            ```yaml
            spec:
              containers:
              # ...
                volumeMounts:
                - name: certs-volume
                  mountPath: /etc/ssl/private
                  readOnly: true # Secret'lar genellikle salt okunur mount edilir
              volumes:
              - name: certs-volume
                secret:
                  secretName: tls-certs
            ```
    *   **Use Case 3: Docker Image Pull Secret Kullanma**
        *   **Senaryo:** `registry.example.io` adresindeki özel bir Docker registry'den imaj çekmek için `my-registry-key` adında bir `imagePullSecret` oluşturun ve bunu `app-from-private-repo-pod` adlı Pod'da kullanın. (Kullanıcı adı: `deploy_user`, şifre: `RegistryP@ssw0rd!`)
        *   **Çözüm Yaklaşımı:** `kubectl create secret docker-registry` ile secret oluşturulur, sonra Pod manifestosunda `imagePullSecrets` alanına eklenir.
        *   **Efektif Çözüm:**
            1.  `kubectl create secret docker-registry my-registry-key --docker-server=registry.example.io --docker-username=deploy_user --docker-password='RegistryP@ssw0rd!' --docker-email=deploy_user@example.com -n <namespace>`
            2.  Pod YAML'ında:
                ```yaml
                spec:
                  containers:
                  - name: private-app
                    image: registry.example.io/my-app:latest
                  imagePullSecrets:
                  - name: my-registry-key
                ```
*   **SecurityContexts (Güvenlik Bağlamları)**
    *   **Çalışma Mantığı Notu:** Hem Pod seviyesinde (`spec.securityContext`) hem de Container seviyesinde (`spec.containers[].securityContext`) tanımlanabilir. Container seviyesindeki ayarlar, Pod seviyesindeki ayarları ezer (override eder) veya onlara eklenir. Örneğin, Pod seviyesinde `runAsUser: 1000` varsa ve bir container'da `runAsUser: 2000` varsa, o container 2000 ID'si ile çalışır.
    *   **Use Case 1: Pod Seviyesinde `runAsUser`, `runAsGroup`, `fsGroup`**
        *   **Senaryo:** Bir Pod'daki tüm container'ların kullanıcı ID 1001, grup ID 2002 ile çalışmasını ve mount edilen volume'lerin dosya sistemi sahipliğinin grup ID 3003 olmasını sağlayın.
        *   **Çözüm Yaklaşımı:** Pod manifestosunda `spec.securityContext` kullanılır.
        *   **Efektif Çözüm:** Pod YAML'ında:
            ```yaml
            spec:
              securityContext:
                runAsUser: 1001
                runAsGroup: 2002
                fsGroup: 3003
              # ... containers
            ```
    *   **Use Case 2: Container Seviyesinde `readOnlyRootFilesystem`, `allowPrivilegeEscalation: false`, `capabilities`**
        *   **Senaryo:** Bir Pod içindeki `hardened-container` adlı container'ın root dosya sisteminin salt okunur olmasını, yetki yükseltmesine izin verilmemesini ve sadece `NET_BIND_SERVICE` ile `SYS_TIME` yeteneklerine sahip olmasını, diğer tüm Linux yeteneklerinin düşürülmesini sağlayın.
        *   **Çözüm Yaklaşımı:** Container tanımında `securityContext` kullanılır.
        *   **Efektif Çözüm:** Pod YAML'ında, ilgili container altında:
            ```yaml
            securityContext:
              readOnlyRootFilesystem: true
              allowPrivilegeEscalation: false
              capabilities:
                drop:
                - ALL
                add:
                - NET_BIND_SERVICE
                - SYS_TIME
            ```
*   **Resource Requirements (Kaynak Gereksinimleri)**
    *   **Çalışma Mantığı Notu:**
        *   **Requests:** Pod zamanlanırken (scheduling) dikkate alınır. Node'da yeterli *istenmemiş* (allocatable) kaynak varsa Pod o node'a atanır. Pod'un çalışması için garanti edilen minimum kaynak miktarıdır.
        *   **Limits:** Pod'un çalışma zamanında kullanabileceği maksimum kaynak miktarıdır. Aşarsa, CPU throttle edilir (yavaşlatılır), bellek aşımında Pod OOMKilled (Out Of Memory Killed) olabilir.
        *   **QoS Sınıfları (Quality of Service):**
            *   `Guaranteed`: Pod'daki her container için hem CPU hem de bellek için request ve limit belirtilmiş VE her kaynak için request == limit ise. En yüksek öncelikli Pod'lardır, kaynak sıkıntısında en son bunlar etkilenir.
            *   `Burstable`: Pod'daki en az bir container için CPU veya bellek request'i belirtilmiş ama Guaranteed koşullarını sağlamıyorsa (örn: limitler request'lerden yüksek veya bazı container'larda sadece request var).
            *   `BestEffort`: Pod'daki hiçbir container için request veya limit belirtilmemişse. En düşük önceliklidir, kaynak sıkıntısında ilk bunlar sonlandırılır veya kaynakları kısıtlanır.
    *   **Use Case:** Bir Deployment'taki `worker-container` adlı container için CPU isteğini "250m" (0.25 CPU), CPU limitini "500m", bellek isteğini "128Mi", bellek limitini "256Mi" olarak ayarlayın.
    *   **Efektif Çözüm (Imperative):**
        `kubectl set resources deployment/<deployment-name> -c worker-container --requests=cpu=250m,memory=128Mi --limits=cpu=500m,memory=256Mi -n <namespace>`
        Veya Pod/Deployment YAML'ında, ilgili container altında:
        ```yaml
        resources:
          requests:
            memory: "128Mi"
            cpu: "250m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        ```
*   **Commands and Arguments (Komutlar ve Argümanlar)**
    *   **Çalışma Mantığı Notu:**
        *   Docker imajlarında `ENTRYPOINT` ve `CMD` bulunur.
        *   Kubernetes'te `command` alanı Docker'ın `ENTRYPOINT`'ini ezer.
        *   Kubernetes'te `args` alanı Docker'ın `CMD`'sini ezer.
        *   Eğer `command` belirtilmiş ama `args` belirtilmemişse, imajın orijinal `CMD`'si yok sayılır.
        *   Eğer sadece `args` belirtilmişse, imajın orijinal `ENTRYPOINT`'i bu argümanlarla çalışır.
    *   **Use Case:** Bir Pod'daki `my-app-container` adlı container'ın (imajı `mycustomapp`) varsayılan Docker imajı `ENTRYPOINT`'ini `/app/start-server.sh` komutuyla ve `CMD` argümanlarını `["--config=/etc/app/config.json", "--loglevel=info"]` ile override edin.
    *   **Çözüm Yaklaşımı:** Container tanımında `command` ve `args` alanları kullanılır.
    *   **Efektif Çözüm:** Pod YAML'ında, ilgili container altında:
        ```yaml
        command: ["/app/start-server.sh"]
        args: ["--config=/etc/app/config.json", "--loglevel=info"]
        ```
*   **Taints and Tolerations (Lekeler ve Toleranslar)**
    *   **Çalışma Mantığı Notu:**
        *   Taint'ler node'lara uygulanır ve Pod'ların o node'a zamanlanmasını "iter" (eğer Pod'un toleransı yoksa).
        *   `NoSchedule`: Yeni Pod'lar bu node'a zamanlanmaz, ancak node üzerinde zaten çalışan ve bu taint'e toleransı olmayan Pod'lar çalışmaya devam eder.
        *   `PreferNoSchedule`: Kubernetes, bu taint'e toleransı olmayan Pod'ları bu node'a zamanlamamaya çalışır, ancak başka uygun node yoksa zamanlayabilir.
        *   `NoExecute`: Yeni Pod'lar bu node'a zamanlanmaz VE node üzerinde zaten çalışan ve bu taint'e toleransı olmayan Pod'lar tahliye edilir (evicted). `tolerationSeconds` ile Pod'un tahliye edilmeden önce ne kadar süre tolerans göstereceği belirlenebilir.
        *   Toleranslar Pod'lara eklenir ve belirli taint'lere sahip node'lara zamanlanmalarını "çeker".
        *   Bir Pod'un bir node'a zamanlanabilmesi için, node'daki tüm `NoSchedule` ve `NoExecute` taint'lerine tolerans göstermesi gerekir (veya taint'in `PreferNoSchedule` olması ve başka seçenek olmaması durumu). Node Selector/Affinity kuralları da ayrıca karşılanmalıdır.
    *   **Use Case:** `special-node-01` adlı node'a `workload_type=batch_processing:NoExecute` taint'i ekleyin. Bu node'da sadece bu taint'e toleransı olan bir Pod (`batch-job-pod`) çalıştırın ve Pod'un tahliye edilmeden önce 60 saniye beklemesini sağlayın.
    *   **Efektif Çözüm:**
        1.  `kubectl taint node special-node-01 workload_type=batch_processing:NoExecute`
        2.  `batch-job-pod` YAML'ında:
            ```yaml
            spec:
              tolerations:
              - key: "workload_type"
                operator: "Equal"
                value: "batch_processing"
                effect: "NoExecute"
                tolerationSeconds: 60
              # ... containers
            ```
*   **Node Selectors (Düğüm Seçiciler)**
    *   **Çalışma Mantığı Notu:**
        *   En basit node seçme yöntemidir. Pod manifestosunda belirtilen etiket-değer çiftlerinin *hepsi* node'daki etiketlerle eşleşiyorsa Pod o node'a zamanlanır (AND mantığı).
        *   Eğer bir Pod'da `nodeSelector` tanımlı değilse, herhangi bir node'a (taint/toleration engeli yoksa ve node affinity kuralları izin veriyorsa) zamanlanabilir. Node'daki etiketlerin bu durumda Pod'un zamanlanması açısından bir önemi yoktur (yani Pod, etiketsiz bir node'a da gidebilir, etiketli bir node'a da).
    *   **Use Case:** `compute-node-05` adlı node'a `accelerator=nvidia-tesla-v100` etiketini ekleyin. Bir Pod'un (`ml-training-pod`) sadece `accelerator=nvidia-tesla-v100` etiketine *ve* `region=eu-central-1` etiketine sahip node'larda çalışmasını sağlayın.
    *   **Efektif Çözüm:**
        1.  Node'a etiket ekleme: `kubectl label node compute-node-05 accelerator=nvidia-tesla-v100` (ve `region` etiketinin de node'da olduğunu varsayalım veya ekleyelim: `kubectl label node compute-node-05 region=eu-central-1`)
        2.  `ml-training-pod` YAML'ında:
            ```yaml
            spec:
              nodeSelector:
                accelerator: nvidia-tesla-v100
                region: eu-central-1
              # ... containers
            ```
*   **Node Affinity (Düğüm Eğilimi)**
    *   **Çalışma Mantığı Notu:** `nodeSelector`'dan daha esnek ve ifade gücü yüksek kurallar sunar (`In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt` operatörleri). İki türü vardır:
        *   `requiredDuringSchedulingIgnoredDuringExecution`: Zamanlama sırasında kural *kesinlikle* karşılanmalı. Pod çalışmaya başladıktan sonra node etiketleri değişse bile (kural artık karşılanmasa da) Pod o node'da çalışmaya devam eder.
        *   `preferredDuringSchedulingIgnoredDuringExecution`: Sistem kuralı karşılamaya çalışır ama garanti etmez. Karşılanamasa bile Pod uygun başka bir node'a zamanlanabilir. `weight` alanı ile tercih sıralaması yapılabilir.
    *   **Use Case:** Bir Pod'un, `disk_type` etiketi `ssd` VEYA `nvme` olan node'larda çalışmasını *tercih edin* (`preferredDuringSchedulingIgnoredDuringExecution`) ve `cpu_arch` etiketi `amd64` olan node'larda çalışmasını *zorunlu kılın* (`requiredDuringSchedulingIgnoredDuringExecution`).
    *   **Efektif Çözüm:** Pod YAML'ında:
        ```yaml
        spec:
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                - matchExpressions:
                  - key: cpu_arch
                    operator: In
                    values:
                    - amd64
              preferredDuringSchedulingIgnoredDuringExecution:
              - weight: 100 # 1-100 arası bir değer, yüksek olan daha çok tercih edilir
                preference:
                  matchExpressions:
                  - key: disk_type
                    operator: In
                    values:
                    - ssd
                    - nvme
          # ... containers
        ```
*   **Pod Affinity and Anti-Affinity (Pod Eğilimi ve Karşı Eğilimi)**
    *   **Çalışma Mantığı Notu:** Pod'ların birbirlerine göre (aynı node'da, farklı node'larda, aynı zone/rack'te vb.) nasıl konumlandırılacağını belirler. `topologyKey` alanı, bu "yakınlık" veya "uzaklık" kavramının hangi topoloji seviyesinde (örn: `kubernetes.io/hostname` node seviyesi, `topology.kubernetes.io/zone` zone seviyesi) değerlendirileceğini belirtir.
    *   **Use Case (Pod Anti-Affinity - Required):** `critical-db-pod` adlı bir Pod'un, `app=critical-db` etiketine sahip başka hiçbir Pod ile aynı node üzerinde *çalışmamasını* zorunlu kılın (yüksek erişilebilirlik için).
    *   **Efektif Çözüm:** `critical-db-pod` YAML'ında:
        ```yaml
        spec:
          affinity:
            podAntiAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
              - labelSelector:
                  matchExpressions:
                  - key: app
                    operator: In
                    values:
                    - critical-db
                topologyKey: "kubernetes.io/hostname"
          # ... containers (ve Pod'un kendisine de app=critical-db etiketi verilmeli)
        ```

### MultiContainer Pods (Çoklu Konteyner Pod'ları)

*   **Sidecar Pattern**
    *   **Use Case:** Ana uygulama container'ı (`main-service`) `/app/data` dizinine veri yazarken, bir `backup-agent` sidecar container'ı periyodik olarak bu dizindeki verileri uzak bir depolama alanına yedeklesin. İki container arasında veri paylaşımı için bir `emptyDir` veya `PersistentVolumeClaim` kullanılabilir.
    *   **Çözüm Yaklaşımı:** Pod manifestosuna ikinci bir container (sidecar) ve paylaşımlı volume eklenir.
    *   **Efektif Çözüm (emptyDir ile):**
        ```yaml
        spec:
          containers:
          - name: main-service
            image: my-main-service-image
            volumeMounts:
            - name: shared-data-volume
              mountPath: /app/data
          - name: backup-agent
            image: my-backup-agent-image
            # Backup agent'ın periyodik çalışma mantığı container içinde olmalı
            volumeMounts:
            - name: shared-data-volume
              mountPath: /data-to-backup # Farklı mount path olabilir
              readOnly: true # Eğer sadece okuyacaksa
          volumes:
          - name: shared-data-volume
            emptyDir: {}
        ```
*   **Init Containers**
    *   **Çalışma Mantığı Notu:** Init container'lar, ana uygulama container'ları başlamadan önce *sırayla* çalışır ve her birinin başarıyla (`exit code 0`) tamamlanması gerekir. Bir init container başarısız olursa, Pod'un `restartPolicy`'sine göre yeniden denenir. Ana container'lar ancak tüm init container'lar başarıyla tamamlandıktan sonra başlar. Kaynak limitleri ana container'lardan ayrı olarak değerlendirilir.
    *   **Use Case:** Ana uygulama (`my-web-app`) başlamadan önce, `/app/config/initial_settings.json` dosyasının bir ConfigMap'ten (`app-initial-config`) kopyalanmasını sağlayan bir init container (`config-copier`) oluşturun. Paylaşımlı bir `emptyDir` volume kullanın.
    *   **Efektif Çözüm:**
        ```yaml
        spec:
          initContainers:
          - name: config-copier
            image: busybox
            command: ['sh', '-c', 'cp /configmap-mount/initial_settings.json /app-config-volume/initial_settings.json']
            volumeMounts:
            - name: configmap-volume # ConfigMap'i mount etmek için
              mountPath: /configmap-mount
            - name: shared-app-config # Paylaşımlı emptyDir
              mountPath: /app-config-volume
          containers:
          - name: my-web-app
            image: my-web-app-image
            volumeMounts:
            - name: shared-app-config
              mountPath: /app/config # Ana container buradan okuyacak
          volumes:
          - name: configmap-volume
            configMap:
              name: app-initial-config # Bu ConfigMap'te initial_settings.json anahtarı olmalı
              items:
              - key: initial_settings.json
                path: initial_settings.json
          - name: shared-app-config
            emptyDir: {}
        ```
*   **Adapter and Ambassador Patterns** (Kavramsal olarak bilinmeli)
    *   **Adapter:** Mevcut bir uygulamanın arayüzünü, sistemin beklediği standart bir arayüze dönüştüren bir sidecar. Örneğin, eski bir uygulamanın loglarını standart bir formata çevirip stdout'a yazmak.
    *   **Ambassador:** Dış servislerle olan iletişimi basitleştiren bir sidecar. Uygulamanın `localhost` üzerinden ambassador'a bağlanmasını sağlar, ambassador ise servis bulma, yeniden deneme, devre kesici gibi karmaşık işlemleri halleder.

### Observability (Gözlemlenebilirlik)

*   **Readiness, Liveness, Startup Probes**
    *   **Çalışma Mantığı Notu (Probe Parametreleri):**
        *   `initialDelaySeconds`: Pod başladıktan sonra probe'un ilk kez çalıştırılmasından önceki saniye cinsinden gecikme. Uygulamanın başlangıç süresine göre ayarlanmalı.
        *   `periodSeconds`: Probe'un ne sıklıkta (saniye cinsinden) çalıştırılacağı.
        *   `timeoutSeconds`: Probe'un yanıt vermesi için beklenecek saniye cinsinden süre. Bu süre içinde yanıt gelmezse probe başarısız sayılır.
        *   `successThreshold`: Başarısız bir probe'dan sonra tekrar başarılı sayılması için gereken ardışık başarılı probe sayısı (varsayılan 1).
        *   `failureThreshold`: Başarılı bir probe'dan sonra başarısız sayılması için gereken ardışık başarısız probe sayısı (varsayılan 3). Liveness probe için bu eşiğe ulaşılınca container yeniden başlatılır. Readiness probe için Pod, Service endpoint'lerinden çıkarılır (trafik almaz). Startup probe için bu eşiğe ulaşılınca container yeniden başlatılır.
        *   **Startup Probe:** Liveness ve Readiness probe'larından önce çalışır. Startup probe başarılı olana kadar diğer probe'lar devreye girmez. Yavaş başlayan uygulamalar için kullanışlıdır, böylece liveness probe'u uygulamanın gerçekten çökmesiyle başlangıçtaki yavaşlığını ayırt edebilir.
    *   **Use Case (Liveness - HTTP GET):** Bir web sunucusu container'ının `/healthz` endpoint'ine (port 80) HTTP GET isteği göndererek canlı olup olmadığını kontrol eden bir liveness probe tanımlayın. Başlangıçta 15 saniye gecikme, her 20 saniyede bir kontrol, 3 başarısız denemeden sonra container'ı yeniden başlatın.
    *   **Efektif Çözüm:** Pod YAML'ında, ilgili container altında:
        ```yaml
        livenessProbe:
          httpGet:
            path: /healthz
            port: 80 # Veya containerPort adı
          initialDelaySeconds: 15
          periodSeconds: 20
          failureThreshold: 3
        ```
    *   **Use Case (Readiness - TCP Socket):** Bir uygulamanın 3306 portunda TCP bağlantısı kabul edip etmediğini kontrol eden bir readiness probe tanımlayın.
    *   **Efektif Çözüm:** Pod YAML'ında, ilgili container altında:
        ```yaml
        readinessProbe:
          tcpSocket:
            port: 3306
          initialDelaySeconds: 5
          periodSeconds: 10
        ```
    *   **Use Case (Startup - Exec Command):** Yavaş başlayan bir uygulama için, `/app/check_boot.sh` script'ini çalıştırarak başlangıç kontrolü yapan bir startup probe tanımlayın. Script başarılı olana kadar (en fazla 30 deneme, her 10 saniyede bir) bekleyin.
    *   **Efektif Çözüm:** Pod YAML'ında, ilgili container altında:
        ```yaml
        startupProbe:
          exec:
            command:
            - /app/check_boot.sh
          failureThreshold: 30
          periodSeconds: 10
        ```
*   **Container Logging (`kubectl logs`)**
    *   **Use Case 1:** `my-app-pod-xyz12` adlı Pod'un `main-app-container` adlı container'ının son 10 log satırını görüntüleyin ve logları canlı olarak takip edin.
    *   **Çözüm Yaklaşımı:** `kubectl logs <pod-name> -c <container-name> --tail=10 -f`
    *   **Efektif Çözüm:** `kubectl logs my-app-pod-xyz12 -c main-app-container --tail=10 -f -n <namespace>`
*   **Monitor and Debug Applications**
    *   **Use Case (`kubectl describe`):** `failing-deployment` adlı bir Deployment neden istediği sayıda Pod oluşturamıyor? `Events` bölümünü inceleyin.
    *   **Çözüm Yaklaşımı:** `kubectl describe deployment <deployment-name>`
    *   **Efektif Çözüm:** `kubectl describe deployment failing-deployment -n <namespace>`

### POD Design (POD Tasarımı)

*   **Labels, Selectors and Annotations (Etiketler, Seçiciler ve Ek Açıklamalar)**
    *   **Çalışma Mantığı Notu (Selectors):**
        *   Kubernetes kaynaklarını (örn: Deployment'ın Pod'ları, Service'in Pod'ları) seçmek için kullanılır.
        *   **Eşitlik Bazlı (Equality-based):** `matchLabels` (YAML'da) veya `kubectl get pods -l key1=value1,key2=value2` (komut satırında). Belirtilen tüm etiket-değer çiftleri tam olarak eşleşmelidir (AND mantığı).
        *   **Küme Bazlı (Set-based):** `matchExpressions` (YAML'da). Daha karmaşık seçimler sağlar:
            *   `key: environment, operator: In, values: [production, staging]` (environment etiketi production VEYA staging olanlar)
            *   `key: tier, operator: NotIn, values: [cache]` (tier etiketi cache olmayanlar)
            *   `key: partition, operator: Exists` (partition etiketi olanlar)
            *   `key: legacy, operator: DoesNotExist` (legacy etiketi olmayanlar)
        *   Bir selector içinde birden fazla `matchExpressions` varsa, bunlar da AND mantığıyla birleştirilir.
    *   **Use Case (Annotation ile Sürüm Bilgisi):** `my-critical-app-deployment` adlı Deployment'a `build_id="jenkins-job-572"` ve `git_commit_sha="a1b2c3d4e5f6"` şeklinde ek açıklamalar (annotations) ekleyin.
    *   **Çözüm Yaklaşımı:** `kubectl annotate deployment <deployment-name> <annotation-key>=<annotation-value>`
    *   **Efektif Çözüm:**
        `kubectl annotate deployment my-critical-app-deployment build_id="jenkins-job-572" git_commit_sha="a1b2c3d4e5f6" -n <namespace> [--overwrite]`
*   **Deployments**
    *   **Use Case (Rolling Update Detayları ve Durum Kontrolü):** `frontend-app` Deployment'ının imajını `myrepo/frontend:v1.2.0` olarak güncelleyin. Güncelleme sırasında en fazla 1 Pod'un kullanılamaz (`maxUnavailable`) ve istenen replika sayısının en fazla %20 üzerinde Pod (`maxSurge`) olmasını sağlayın. Ayrıca, her yeni Pod'un hazır (`ready`) olması için minimum 20 saniye (`minReadySeconds`) bekleyin. Güncelleme durumunu izleyin ve tamamlandığında doğrulayın.
    *   **Çözüm Yaklaşımı:** `kubectl set image`, Deployment YAML'ında `strategy.rollingUpdate` ve `minReadySeconds` ayarları, `kubectl rollout status`.
    *   **Efektif Çözüm:**
        1.  Deployment YAML'ını düzenle (`kubectl edit deployment frontend-app -n <namespace>`):
            ```yaml
            spec:
              minReadySeconds: 20
              strategy:
                type: RollingUpdate
                rollingUpdate:
                  maxUnavailable: 1 # veya "25%" gibi bir yüzde
                  maxSurge: "20%"   # veya 2 gibi bir sayı
            ```
        2.  `kubectl set image deployment/frontend-app <container-name>=myrepo/frontend:v1.2.0 -n <namespace> --record` (`--record` artık deprecated, annotation ile yapılabilir veya history'den takip edilir)
        3.  Güncelleme durumunu izle: `kubectl rollout status deployment/frontend-app -n <namespace>`
        4.  Güncelleme geçmişini gör: `kubectl rollout history deployment/frontend-app -n <namespace>`
    *   **Canary Deployment (Basit Yaklaşım)**
        *   **Use Case:** `main-service` Deployment'ı (10 replika, `version: stable` etiketi) çalışırken, yeni bir sürümü (`version: canary`, imaj `main-service:new`) 1 replika ile deploy edin. `main-service-access` adlı Service her iki sürümü de (`app: main-service` etiketi ile) hedeflesin.
        *   **Çözüm Yaklaşımı:**
            1.  Yeni sürüm için ayrı bir Deployment (`main-service-canary`) oluşturun (1 replika, `app: main-service`, `version: canary` etiketleri).
            2.  Mevcut Service'in (`main-service-access`) selector'ının her iki Deployment'ı da kapsadığından emin olun (örn: `app: main-service`). Trafik, replika sayılarına göre (10:1) dağılacaktır.
            3.  Canary sürümü izleyin. Başarılıysa, canary Deployment'ını scale up edip stable Deployment'ını scale down edin (veya stable'ı yeni imajla güncelleyip canary'yi silin). Sorun varsa, canary Deployment'ını silin veya scale down edin.
        *   **Efektif Çözüm (Canary Deployment YAML Parçası):**
            ```yaml
            apiVersion: apps/v1
            kind: Deployment
            metadata:
              name: main-service-canary
            spec:
              replicas: 1
              selector:
                matchLabels:
                  app: main-service
                  version: canary
              template:
                metadata:
                  labels:
                    app: main-service
                    version: canary
                spec:
                  containers:
                  - name: main-container
                    image: myrepo/main-service:new # Yeni sürüm imajı
                    # ...
            ```
            Mevcut Service (`main-service-access`) YAML'ında selector:
            ```yaml
            selector:
              app: main-service # Hem stable hem canary'yi seçer
            ```
    *   **Blue/Green Deployment (Basit Yaklaşım)**
        *   **Use Case:** `webapp:blue` (imaj `webapp:v1.0`, `version: blue` etiketi) çalışan sürümken, `webapp:green` (imaj `webapp:v1.1`, `version: green` etiketi) adlı yeni sürümü ayrı bir Deployment ile tam kapasitede (blue ile aynı replika sayısı) deploy edin. `webapp-router-svc` adlı Service'i, testler tamamlandıktan sonra trafiği anında `blue` sürümünden `green` sürümüne yönlendirecek şekilde güncelleyin.
        *   **Çözüm Yaklaşımı:**
            1.  İki ayrı Deployment (`webapp-blue`, `webapp-green`) oluşturun.
            2.  Service (`webapp-router-svc`) başlangıçta `version: blue` etiketli Pod'ları hedefler.
            3.  `webapp-green` Deployment'ı deploy edilir ve test edilir (örn: farklı bir test Service'i veya port-forward ile).
            4.  Geçiş için `webapp-router-svc`'in selector'ı `version: green` olarak güncellenir.
        *   **Efektif Çözüm (Service Güncelleme Adımı):**
            *   Başlangıçta `webapp-router-svc.yaml` selector:
                ```yaml
                selector:
                  app: webapp
                  version: blue
                ```
            *   Green'e geçiş için Service'i patch'leyin:
                `kubectl patch service webapp-router-svc -p '{"spec":{"selector":{"app":"webapp", "version":"green"}}}' -n <namespace>`
                Veya Service YAML'ını düzenleyip `kubectl apply -f webapp-router-svc.yaml -n <namespace>` ile uygulayın.
*   **Jobs and CronJobs**
    *   **Çalışma Mantığı Notu (Job):**
        *   `restartPolicy` Pod template'i içinde `OnFailure` (başarısız olursa Pod içindeki container'lar yeniden başlatılır, Job yeniden dener) veya `Never` (başarısız olursa Pod hata durumunda kalır, Job yeniden dener) olabilir. `Always` olamaz.
        *   `backoffLimit`: Bir Job'un başarısız sayılmadan önce kaç kez yeniden deneneceği (Pod seviyesinde başarısızlıklar için, varsayılan 6).
        *   `activeDeadlineSeconds`: Job'un ne kadar süreyle aktif kalabileceğini belirtir. Bu süre aşılırsa, çalışan tüm Pod'ları sonlandırılır ve Job başarısız olarak işaretlenir.
        *   `ttlSecondsAfterFinished`: Tamamlanmış (başarılı veya başarısız) Job'ların ve onlara ait Pod'ların otomatik olarak silinmeden önce ne kadar süreyle tutulacağını belirtir.
    *   **Use Case (CronJob - Concurrency Policy):** Her 5 dakikada bir çalışan `data-cleanup-cronjob` adlı bir CronJob oluşturun. Eğer önceki Job çalışması hala devam ediyorsa, yeni bir Job başlatılmasın (`concurrencyPolicy: Forbid`). Başarılı Job'ların geçmişi 3, başarısız Job'ların geçmişi 1 olarak tutulsun.
    *   **Efektif Çözüm:** `cleanup-cronjob.yaml`:
        ```yaml
        apiVersion: batch/v1
        kind: CronJob
        metadata:
          name: data-cleanup-cronjob
        spec:
          schedule: "*/5 * * * *"
          concurrencyPolicy: Forbid # Allow (varsayılan), Forbid, Replace
          successfulJobsHistoryLimit: 3
          failedJobsHistoryLimit: 1
          jobTemplate:
            spec:
              template:
                spec:
                  containers:
                  - name: cleanup-container
                    image: my-cleanup-script-runner
                    # ... command/args
                  restartPolicy: OnFailure
        ```
        `kubectl apply -f cleanup-cronjob.yaml -n <namespace>`
*   **PodDisruptionBudgets (PDBs)**
    *   **Use Case:** `app=payment-gateway` etiketli Pod'lardan en az 2 tanesinin her zaman çalışır durumda olmasını (`minAvailable: 2`) garanti eden bir PDB (`payment-pdb`) oluşturun. Bu, gönüllü kesintiler (node bakımı, küme yükseltmesi gibi) sırasında uygulamanın erişilebilirliğini korur.
    *   **Çözüm Yaklaşımı:** PDB manifestosu oluşturulur.
    *   **Efektif Çözüm:** `payment-pdb.yaml`:
        ```yaml
        apiVersion: policy/v1 # Eski versiyonlarda policy/v1beta1 olabilir
        kind: PodDisruptionBudget
        metadata:
          name: payment-pdb
        spec:
          minAvailable: 2 # veya maxUnavailable: 1 (eğer toplam 3 pod varsa)
          selector:
            matchLabels:
              app: payment-gateway
        ```
        `kubectl apply -f payment-pdb.yaml -n <namespace>`
*   **DaemonSets**
    *   **CKAD Kapsamında mı?** Evet.
    *   **Use Case:** Kümedeki `role=worker` etiketine sahip her node'da bir metrik toplama ajanı (`node-exporter`) çalıştıran bir DaemonSet (`metrics-agent-ds`) oluşturun. Master node'larda çalışmasın (master node'ların `node-role.kubernetes.io/master:NoSchedule` taint'i olduğunu varsayalım).
    *   **Çözüm Yaklaşımı:** DaemonSet manifestosu oluşturulur. `spec.template.spec.nodeSelector` ile hedef node'lar seçilir ve `spec.template.spec.tolerations` ile master node taint'ine tolerans gösterilir (eğer tüm node'larda çalışması isteniyorsa) veya gösterilmez (eğer sadece worker'larda çalışması isteniyorsa ve worker'larda bu taint yoksa).
    *   **Efektif Çözüm:** `metrics-agent-ds.yaml`:
        ```yaml
        apiVersion: apps/v1
        kind: DaemonSet
        metadata:
          name: metrics-agent-ds
        spec:
          selector:
            matchLabels:
              name: node-exporter-agent
          template:
            metadata:
              labels:
                name: node-exporter-agent
            spec:
              nodeSelector: # Sadece worker node'ları hedeflemek için
                role: worker
              # Eğer tüm node'larda (master dahil) çalışması istenseydi ve master'da taint olsaydı:
              # tolerations:
              # - key: "node-role.kubernetes.io/master"
              #   operator: "Exists"
              #   effect: "NoSchedule"
              containers:
              - name: node-exporter-container
                image: prom/node-exporter:latest
                ports:
                - containerPort: 9100
                  name: metrics
        ```
        `kubectl apply -f metrics-agent-ds.yaml -n <namespace>`
*   **StatefulSets**
    *   **CKAD Kapsamında mı?** Evet.
    *   **Use Case:** Her bir replikası için kararlı, benzersiz bir ağ tanımlayıcısına (örn: `zk-0`, `zk-1`) ve kararlı, kalıcı depolamaya (her biri için ayrı bir PVC) ihtiyaç duyan bir Zookeeper ensemble (`zk-ensemble`) için 3 replikalı bir StatefulSet (`zookeeper-sts`) oluşturun. Pod'ların birbirini bulabilmesi için `zookeeper-headless` adlı bir Headless Service kullanılacak. Güncellemeler sıralı (rolling update) ve particion bazlı yapılsın.
    *   **Çözüm Yaklaşımı:**
        1.  Headless Service oluşturulur.
        2.  StatefulSet manifestosu oluşturulur. `volumeClaimTemplates` ile her Pod için dinamik olarak PVC oluşturulur. `updateStrategy` tanımlanır.
    *   **Efektif Çözüm:**
        *   `zookeeper-headless.yaml`:
            ```yaml
            apiVersion: v1
            kind: Service
            metadata:
              name: zookeeper-headless
              labels:
                app: zookeeper
            spec:
              clusterIP: None
              selector:
                app: zookeeper # StatefulSet Pod'larının etiketiyle eşleşmeli
              ports:
              - port: 2181
                name: client
              - port: 2888
                name: server
              - port: 3888
                name: leader-election
            ```
        *   `zookeeper-sts.yaml`:
            ```yaml
            apiVersion: apps/v1
            kind: StatefulSet
            metadata:
              name: zookeeper-sts
            spec:
              serviceName: "zookeeper-headless"
              replicas: 3
              selector:
                matchLabels:
                  app: zookeeper
              template:
                metadata:
                  labels:
                    app: zookeeper
                spec:
                  containers:
                  - name: zookeeper-container
                    image: zookeeper:3.8 # Örnek imaj
                    ports:
                    - containerPort: 2181
                      name: client
                    - containerPort: 2888
                      name: server
                    - containerPort: 3888
                      name: leader-election
                    volumeMounts:
                    - name: data-volume # volumeClaimTemplates'teki name ile eşleşmeli
                      mountPath: /data/zookeeper
              volumeClaimTemplates:
              - metadata:
                  name: data-volume
                spec:
                  accessModes: [ "ReadWriteOnce" ]
                  storageClassName: "standard" # Uygun bir StorageClass
                  resources:
                    requests:
                      storage: 5Gi
              updateStrategy: # Örnek güncelleme stratejisi
                type: RollingUpdate
                rollingUpdate:
                  partition: 0 # Tüm podlar güncellenir. Daha yüksek bir sayı, o ordinal'den sonrakileri günceller.
            ```
        1.  `kubectl apply -f zookeeper-headless.yaml -n <namespace>`
        2.  `kubectl apply -f zookeeper-sts.yaml -n <namespace>`

### Services and Networking (Servisler ve Ağ İletişimi)

*   **Service Tipleri:**
    *   **Use Case (ExternalName Service):** Küme içindeki uygulamaların, `external-db.example.com` adresindeki harici bir veritabanına `internal-db-alias` adıyla erişmesini sağlayan bir Service oluşturun.
    *   **Çözüm Yaklaşımı:** `type: ExternalName` ve `externalName` alanı kullanılır.
    *   **Efektif Çözüm:** `externalname-svc.yaml`:
        ```yaml
        apiVersion: v1
        kind: Service
        metadata:
          name: internal-db-alias
        spec:
          type: ExternalName
          externalName: external-db.example.com
        ```
        `kubectl apply -f externalname-svc.yaml -n <namespace>`
*   **Ingress Resource ve Ingress Controller**
    *   **Use Case (TLS Sonlandırma ve Host Bazlı Yönlendirme):** `app.example.com` adresine gelen HTTPS trafiğini `app-service`'e (port 80), `api.example.com` adresine gelen HTTPS trafiğini ise `api-service`'e (port 8080) yönlendiren bir Ingress (`secure-ingress`) oluşturun. TLS sonlandırması için `example-tls-secret` adlı Secret'ı kullanın.
    *   **Çözüm Yaklaşımı:** Ingress manifestosunda `tls` bölümü ve birden fazla `rules.host` tanımlanır. (Ingress controller ve TLS secret'ın önceden var olduğu varsayılır).
    *   **Efektif Çözüm:** `secure-ingress.yaml`:
        ```yaml
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        metadata:
          name: secure-ingress
          # annotations:
          #   nginx.ingress.kubernetes.io/ssl-redirect: "true" # HTTP'yi HTTPS'e yönlendirme (controller'a bağlı)
        spec:
          tls:
          - hosts:
            - app.example.com
            - api.example.com
            secretName: example-tls-secret # TLS sertifikasını içeren Secret
          rules:
          - host: app.example.com
            http:
              paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: app-service
                    port:
                      number: 80
          - host: api.example.com
            http:
              paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: api-service
                    port:
                      number: 8080
        ```
        `kubectl apply -f secure-ingress.yaml -n <namespace>`
*   **Network Policies (Ağ Politikaları)**
    *   **Çalışma Mantığı Notu:**
        *   Bir namespace'de herhangi bir NetworkPolicy yoksa veya bir Pod herhangi bir NetworkPolicy tarafından seçilmiyorsa (`podSelector`'ı o Pod ile eşleşmiyorsa), o Pod için tüm gelen (ingress) ve giden (egress) trafik serbesttir.
        *   Bir Pod, en az bir NetworkPolicy tarafından `podSelector` ile seçiliyorsa, o Pod'a sadece o NetworkPolicy (veya onu seçen diğer NetworkPolicy'ler) tarafından açıkça izin verilen trafik girebilir/çıkabilir. İzin verilmeyen her şey engellenir (varsayılan deny).
        *   `policyTypes: ["Ingress"]` sadece gelen trafiği, `policyTypes: ["Egress"]` sadece giden trafiği, her ikisi de varsa veya hiç belirtilmemişse (ve ingress/egress kuralları varsa) her ikisini de etkiler.
        *   Boş bir `ingress: {}` veya `egress: {}` kuralı, o yöndeki tüm trafiğe izin verir (eğer Pod bir policy tarafından seçiliyorsa).
        *   Boş bir `ingress: []` veya `egress: []` (veya ilgili bölümün hiç olmaması, ama `policyTypes`'ın o yönü içermesi) o yöndeki tüm trafiği engeller.
    *   **Use Case (Egress Kısıtlaması):** `app=frontend` etiketli Pod'ların sadece `app=backend` etiketli Pod'lara 8080 TCP portundan ve `kube-dns` (port 53 UDP/TCP) servisine giden trafiğine izin verin. Diğer tüm giden trafik engellensin.
    *   **Efektif Çözüm:** `frontend-egress-policy.yaml`:
        ```yaml
        apiVersion: networking.k8s.io/v1
        kind: NetworkPolicy
        metadata:
          name: frontend-egress-policy
        spec:
          podSelector:
            matchLabels:
              app: frontend
          policyTypes:
          - Egress
          egress:
          - to: # Backend'e izin
            - podSelector:
                matchLabels:
                  app: backend
            ports:
            - protocol: TCP
              port: 8080
          - to: # DNS'e izin (kube-system namespace'indeki kube-dns için)
            - namespaceSelector:
                matchLabels:
                  kubernetes.io/metadata.name: kube-system # veya name: kube-system (eski sürümlerde)
              podSelector:
                matchLabels:
                  k8s-app: kube-dns
            ports:
            - protocol: UDP
              port: 53
            - protocol: TCP
              port: 53
        ```
        `kubectl apply -f frontend-egress-policy.yaml -n <namespace>`

### State Persistence (Durum Kalıcılığı)

*   **Volumes (Birimler)**
    *   **Use Case (`downwardAPI`):** Bir Pod'a kendi adını, namespace'ini ve CPU limitini ortam değişkenleri olarak ve etiketlerini `/etc/podinfo` altına dosya olarak geçirin.
    *   **Çözüm Yaklaşımı:** `downwardAPI` volume tipi ve `env.valueFrom.fieldRef/resourceFieldRef` kullanılır.
    *   **Efektif Çözüm:** Pod YAML'ında:
        ```yaml
        spec:
          containers:
          - name: downward-api-test
            image: busybox
            command: ["sh", "-c", "printenv POD_NAME POD_NAMESPACE CPU_LIMIT && ls -l /etc/podinfo && sleep 3600"]
            env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: CPU_LIMIT
              valueFrom:
                resourceFieldRef:
                  containerName: downward-api-test # Kendi container adı
                  resource: limits.cpu
            volumeMounts:
            - name: podinfo-volume
              mountPath: /etc/podinfo
          volumes:
          - name: podinfo-volume
            downwardAPI:
              items:
              - path: "labels" # Dosya adı
                fieldRef:
                  fieldPath: metadata.labels
              - path: "annotations"
                fieldRef:
                  fieldPath: metadata.annotations
        ```
*   **Persistent Volumes (PVs) ve Persistent Volume Claims (PVCs)**
    *   (PV oluşturma genellikle CKAD kapsamında değildir, ancak PVC oluşturma ve kullanma önemlidir.)
*   **StorageClasses (Depolama Sınıfları)**
    *   **Use Case:** Mevcut StorageClass'ları listeleyin ve `fast-storage` adlı bir StorageClass'ın `provisioner` ve `parameters` bilgilerini görüntüleyin.
    *   **Çözüm Yaklaşımı:** `kubectl get sc` ve `kubectl describe sc <sc-name>`
    *   **Efektif Çözüm:**
        1.  `kubectl get storageclass`
        2.  `kubectl describe storageclass fast-storage`
*   **Access Modes (Erişim Modları)**
    *   `ReadWriteOnce` (RWO): Volume tek bir node tarafından okunabilir/yazılabilir olarak mount edilebilir.
    *   `ReadOnlyMany` (ROX): Volume birçok node tarafından salt okunur olarak mount edilebilir.
    *   `ReadWriteMany` (RWX): Volume birçok node tarafından okunabilir/yazılabilir olarak mount edilebilir.
    *   `ReadWriteOncePod` (RWOP): Volume tek bir Pod tarafından okunabilir/yazılabilir olarak mount edilebilir. (Daha yeni bir özellik)
*   **StorageClass, PV, PVC ve Pod'a Ekleme Entegrasyonu**
    *   **Çalışma Mantığı Notu:**
        *   **Dinamik Provizyonlama (En Yaygın Senaryo):**
            1.  Yönetici bir veya daha fazla `StorageClass` tanımlar (örn: `aws-ebs-gp3`, `azure-disk-premium`). Her StorageClass, belirli bir depolama sağlayıcısının (provisioner) nasıl PV oluşturacağını ve hangi parametreleri kullanacağını belirtir.
            2.  Kullanıcı bir `PersistentVolumeClaim` (PVC) oluşturur. PVC'de istediği `storageClassName`'i, erişim modunu (`accessModes`) ve depolama boyutunu (`resources.requests.storage`) belirtir.
            3.  Kubernetes (daha doğrusu StorageClass'ın belirttiği provisioner), bu PVC'ye uygun bir `PersistentVolume` (PV) otomatik olarak oluşturur ve PVC'yi bu PV'ye bağlar (Bound).
            4.  Kullanıcı, Pod manifestosunda bu PVC'yi bir volume olarak referans göstererek kullanır. Pod zamanlandığında, Kubernetes bu volume'ü Pod'un çalışacağı node'a mount eder.
        *   **Statik Provizyonlama:** Yönetici manuel olarak PV'ler oluşturur. Kullanıcı PVC oluşturduğunda, Kubernetes bu PVC'nin gereksinimlerini karşılayan mevcut, bağlanmamış (Available) bir PV bulmaya çalışır.
        *   **CKAD'de Genellikle:** Sizden bir PVC oluşturmanız ve bunu bir Pod'a mount etmeniz istenir. Arka planda bir StorageClass'ın dinamik provizyonlama yapacağı veya sınav ortamında uygun statik PV'lerin önceden tanımlanmış olacağı varsayılır.
    *   **Use Case (Dinamik Provizyonlama ile Entegre Akış):**
        1.  **Senaryo (PVC):** `webapp-content-pvc` adında, 5Gi boyutunda, `ReadWriteOnce` erişim modu isteyen ve (varsa) `managed-premium` StorageClass'ını kullanan bir PVC oluşturun.
        2.  **Senaryo (Pod):** `content-server-pod` adlı bir Pod'a, yukarıda oluşturulan `webapp-content-pvc`'yi `/usr/share/nginx/html` yoluna mount edin.
        *   **Efektif Çözüm:**
            *   `webapp-pvc.yaml`:
                ```yaml
                apiVersion: v1
                kind: PersistentVolumeClaim
                metadata:
                  name: webapp-content-pvc
                spec:
                  accessModes:
                    - ReadWriteOnce
                  storageClassName: managed-premium # Ortamda bu SC'nin olması beklenir
                  resources:
                    requests:
                      storage: 5Gi
                ```
            *   `content-pod.yaml`:
                ```yaml
                apiVersion: v1
                kind: Pod
                metadata:
                  name: content-server-pod
                spec:
                  containers:
                  - name: nginx-server
                    image: nginx
                    ports:
                    - containerPort: 80
                    volumeMounts:
                    - name: web-content-storage
                      mountPath: /usr/share/nginx/html # Nginx'in varsayılan içerik yolu
                  volumes:
                  - name: web-content-storage
                    persistentVolumeClaim:
                      claimName: webapp-content-pvc
                ```
            1.  `kubectl apply -f webapp-pvc.yaml -n <namespace>`
            2.  `kubectl apply -f content-pod.yaml -n <namespace>`

### Security (Güvenlik)

*   **RBAC: ServiceAccounts, Roles, RoleBindings**
    *   **Çalışma Mantığı Notu:**
        *   **Authentication (Kimlik Doğrulama):** "Kimsin?" sorusunun cevabıdır. Kubernetes'te iki tür kullanıcı vardır: Normal Kullanıcılar (insanlar, genellikle dış kimlik sağlayıcıları ile yönetilir) ve ServiceAccount'lar (Pod'lar içinde çalışan süreçler için Kubernetes tarafından yönetilen kimlikler).
        *   **Authorization (Yetkilendirme):** "Ne yapabilirsin?" sorusunun cevabıdır. Kimliği doğrulanmış bir öznenin (kullanıcı, grup, ServiceAccount) hangi kaynaklar üzerinde hangi işlemleri (verbs: get, list, watch, create, update, patch, delete) yapabileceğini belirler. RBAC (Role-Based Access Control) en yaygın yetkilendirme mekanizmasıdır.
        *   **ServiceAccount (SA):** Pod'lar içinde çalışan uygulamaların Kubernetes API sunucusuna erişmesi için bir kimlik sağlar. Her namespace'de varsayılan olarak `default` adında bir ServiceAccount bulunur. Pod'lar, `serviceAccountName` alanında belirtilmezse bu `default` SA'yı kullanır.
        *   **Role (Namespace Kapsamlı):** Bir namespace içindeki API kaynaklarına (örn: pods, configmaps, secrets) erişim kurallarını tanımlar.
        *   **RoleBinding (Namespace Kapsamlı):** Bir Role'ü, aynı namespace içindeki belirli kullanıcılara, gruplara veya ServiceAccount'lara (subjects) bağlar.
        *   **ClusterRole (Küme Kapsamlı):** Küme genelindeki kaynaklara (örn: nodes, persistentvolumes, namespaces) veya *tüm* namespace'lerdeki namespace-kapsamlı kaynaklara (örn: tüm namespace'lerdeki Pod'lar) erişim kurallarını tanımlar.
        *   **ClusterRoleBinding (Küme Kapsamlı):** Bir ClusterRole'ü, küme genelinde kullanıcılara, gruplara veya ServiceAccount'lara bağlar.
        *   CKAD sınavı daha çok namespace kapsamlı Role ve RoleBinding'lere odaklanır.
    *   **Use Case 1: Pod'a Özel ServiceAccount Atama ve Yetkilendirme (ConfigMap Okuma)**
        *   **Senaryo:**
            1.  `config-reader-sa` adında bir ServiceAccount oluşturun.
            2.  Bu ServiceAccount'a, bulunduğu namespace içindeki `app-global-config` adlı belirli bir ConfigMap'i okuma (`get`) yetkisi veren bir Role (`specific-configmap-reader-role`) ve bu Role'ü SA'ya bağlayan bir RoleBinding (`specific-configmap-reader-binding`) oluşturun.
            3.  `config-consuming-app-pod` adlı bir Pod'un bu `config-reader-sa` ServiceAccount'ını kullanmasını sağlayın.
        *   **Çözüm Yaklaşımı:** SA, Role, RoleBinding ve Pod manifestoları oluşturulur. Role'de `resourceNames` kullanılır.
        *   **Efektif Çözüm:**
            1.  `kubectl create serviceaccount config-reader-sa -n <namespace>`
            2.  Role YAML (`specific-configmap-reader-role.yaml`):
                ```yaml
                apiVersion: rbac.authorization.k8s.io/v1
                kind: Role
                metadata:
                  name: specific-configmap-reader-role
                rules:
                - apiGroups: [""] # Core API grubu
                  resources: ["configmaps"]
                  resourceNames: ["app-global-config"] # Sadece bu ConfigMap'e erişim
                  verbs: ["get"]
                ```
                `kubectl apply -f specific-configmap-reader-role.yaml -n <namespace>`
            3.  `kubectl create rolebinding specific-configmap-reader-binding --role=specific-configmap-reader-role --serviceaccount=<namespace>:config-reader-sa -n <namespace>`
            4.  `config-consuming-app-pod.yaml`:
                ```yaml
                apiVersion: v1
                kind: Pod
                metadata:
                  name: config-consuming-app-pod
                spec:
                  serviceAccountName: config-reader-sa
                  containers:
                  - name: app-container
                    image: appropriate/kubectl # Veya kubectl içeren başka bir imaj
                    # Bu komut, SA'nın yetkisi varsa ConfigMap'i getirecektir
                    command: ["kubectl", "get", "configmap", "app-global-config", "-n", "<namespace>", "-o", "yaml"]
                ```
                `kubectl apply -f config-consuming-app-pod.yaml -n <namespace>`
    *   **Use Case 2: ServiceAccount'ın Token'ının Otomatik Mount Edilmemesi**
        *   **Senaryo:** `no-api-access-pod` adlı bir Pod oluşturun. Bu Pod'un ServiceAccount token'ının (`/var/run/secrets/kubernetes.io/serviceaccount/token`) otomatik olarak mount edilmemesini sağlayın. Bu, Pod'un Kubernetes API'sine erişmesi gerekmiyorsa güvenlik açısından iyi bir pratiktir.
        *   **Çözüm Yaklaşımı:** Pod manifestosunda `automountServiceAccountToken: false` ayarlanır. Eğer ServiceAccount seviyesinde de `automountServiceAccountToken: false` ayarlanırsa, o SA'yı kullanan tüm Pod'lar için varsayılan bu olur (Pod seviyesinde override edilebilir).
        *   **Efektif Çözüm:** `no-api-access-pod.yaml`:
            ```yaml
            apiVersion: v1
            kind: Pod
            metadata:
              name: no-api-access-pod
            spec:
              automountServiceAccountToken: false # Pod seviyesinde token mount'unu engelle
              containers:
              - name: my-isolated-container
                image: busybox
                command: ["sleep", "3600"]
            ```
*   **İmaj güvenliği (Image security)** (Kavramsal olarak bilinmeli)
    *   Güvenilir kaynaklardan (örn: resmi Docker Hub repoları, kendi özel güvenli registry'niz) imaj kullanma.
    *   Mümkünse minimal taban imajları (distroless, alpine) tercih etme.
    *   İmajların düzenli olarak zafiyet taramasından geçirilmesi (örn: Trivy, Clair).
    *   Özel registry (private registry) kullanımı ve `imagePullSecrets` ile güvenli erişim.
*   **API Grupları ve Kaynaklara Erişim** (RBAC ile ilgili)
    *   Bir ServiceAccount'ın veya uygulamanın hangi API gruplarına (örn: `""` core için, `apps`, `batch`, `networking.k8s.io`) ve bu gruplar altındaki hangi kaynaklara (pods, services, deployments, ingresses vb.) hangi operasyonları (get, list, watch, create, update, patch, delete) yapabileceğini anlama ve kısıtlama.
*   **Minimum Yetki Prensibi (Principle of Least Privilege)**
    *   Pod'lara, ServiceAccount'lara ve kullanıcılara sadece görevlerini yerine getirebilmeleri için *gerekli olan minimum* yetkileri verme. Örneğin, bir uygulamanın sadece ConfigMap okuması gerekiyorsa, ona ConfigMap yazma veya Secret okuma yetkisi vermemek.
*   **Secrets yönetimi** (Kavramsal olarak bilinmeli, pratikleri ConfigMaps/Secrets altında)
    *   Hassas verilerin (şifreler, API tokenları, TLS sertifikaları) güvenli bir şekilde saklanması ve Pod'lara iletilmesi. Ortam değişkenleri veya volume mount'ları ile.
    *   Secret'ların şifrelenmesi (etcd seviyesinde veya harici KMS ile).

### ResourceQuotas ve LimitRanges

*   **ResourceQuotas**
    *   **Use Case:** `<production-namespace>` için toplamda en fazla 20 Pod, 10 CPU (request), 5 CPU (limit), 32Gi bellek (request) ve 16Gi bellek (limit) kotası tanımlayın. Ayrıca en fazla 5 Service oluşturulabilsin.
    *   **Çözüm Yaklaşımı:** ResourceQuota manifestosu oluşturulur.
    *   **Efektif Çözüm:** `prod-quota.yaml`:
        ```yaml
        apiVersion: v1
        kind: ResourceQuota
        metadata:
          name: prod-ns-quota
          namespace: <production-namespace> # Namespace burada belirtilmeli
        spec:
          hard:
            pods: "20"
            requests.cpu: "10"
            limits.cpu: "5" # Genellikle limit request'ten büyük veya eşit olur, bu örnekte farklı bir senaryo
            requests.memory: "32Gi"
            limits.memory: "16Gi" # Aynı şekilde, limit request'ten küçük olabilir ama dikkatli olunmalı
            services: "5"
            # configmaps: "10"
            # secrets: "15"
        ```
        `kubectl apply -f prod-quota.yaml` (Namespace önceden oluşturulmuş olmalı)
*   **LimitRanges**
    *   **Use Case:** `<development-namespace>` içindeki container'lar için varsayılan CPU isteğini 150m, bellek isteğini 100Mi; varsayılan CPU limitini 300m, bellek limitini 200Mi olarak ayarlayın. Ayrıca, bir container'ın isteyebileceği maksimum bellek 1Gi, minimum bellek ise 50Mi olsun.
    *   **Çözüm Yaklaşımı:** LimitRange manifestosu oluşturulur.
    *   **Efektif Çözüm:** `dev-limitrange.yaml`:
        ```yaml
        apiVersion: v1
        kind: LimitRange
        metadata:
          name: dev-ns-limits
          namespace: <development-namespace> # Namespace burada belirtilmeli
        spec:
          limits:
          - type: Container
            defaultRequest: # Eğer Pod'da request belirtilmemişse bu kullanılır
              cpu: "150m"
              memory: "100Mi"
            default: # Eğer Pod'da limit belirtilmemişse bu kullanılır
              cpu: "300m"
              memory: "200Mi"
            max: # Bir container'ın isteyebileceği maksimum limit
              memory: "1Gi"
            min: # Bir container'ın isteyebileceği minimum request
              memory: "50Mi"
            # maxLimitRequestRatio: # Limit'in request'e oranı için de kısıt konabilir
            #   cpu: "4" # Limit, request'in en fazla 4 katı olabilir
        ```
        `kubectl apply -f dev-limitrange.yaml`
