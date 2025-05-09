### Genel Sinav Stratejileri
*   Namespace/Context: Her zaman doğru namespace'de olduğunuzdan emin olun.
*   Imperative Komutlar: kubectl create/run/expose/set/scale/label/annotate/rollout gibi komutları önceliklendirin.
*   YAML (--dry-run=client -o yaml ve kubectl get ... -o yaml): Imperative komutların yetmediği yerde hızlıca YAML oluşturup düzenleyin.
*   kubectl explain: YAML alanları için vazgeçilmez yardımcınız. kubeclt <command> -h de etkilidir. 

### Configuration (Yapılandırma)

*   **ConfigMaps**
    *   **Use Case 1: Ortam Değişkeni Olarak Kullanma (Tüm Anahtarlar)**
        *   **Senaryo:** `app-config` adında bir ConfigMap oluşturun (`db_host=mysql`, `db_port=3306` verileriyle). Bu ConfigMap'teki tüm anahtar-değer çiftlerini `my-pod` adlı bir Pod'un `main-container` adlı container'ına ortam değişkeni olarak atayın.
        *   **Çözüm Yaklaşımı:** Önce ConfigMap oluşturulur, sonra Pod manifestosunda `envFrom` ve `configMapRef` kullanılır.
        *   **Efektif Çözüm:**
            1.  `kubectl create configmap app-config --from-literal=db_host=mysql --from-literal=db_port=3306 -n <namespace>`
            2.  Pod YAML'ı (`pod.yaml`):
                ```yaml
                apiVersion: v1
                kind: Pod
                metadata:
                  name: my-pod
                spec:
                  containers:
                  - name: main-container
                    image: busybox
                    command: ["sleep", "3600"]
                    envFrom:
                    - configMapRef:
                        name: app-config
                ```
            3.  `kubectl apply -f pod.yaml -n <namespace>`
            4.  Kontrol: `kubectl exec my-pod -n <namespace> -- env | grep DB_`
    *   **Use Case 2: Ortam Değişkeni Olarak Kullanma (Belirli Anahtar)**
        *   **Senaryo:** `app-config` ConfigMap'indeki `db_host` anahtarının değerini `DB_ENDPOINT` adlı bir ortam değişkeni olarak `my-pod`'a atayın.
        *   **Çözüm Yaklaşımı:** Pod manifestosunda `env` altında `valueFrom` ve `configMapKeyRef` kullanılır.
        *   **Efektif Çözüm:** Pod YAML'ı düzenlenir:
            ```yaml
            # ... (yukarıdaki Pod YAML'ının devamı)
            env:
            - name: DB_ENDPOINT
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: db_host
            # ...
            ```
    *   **Use Case 3: Volume Mount Olarak Kullanma**
        *   **Senaryo:** `app-config` ConfigMap'ini `my-pod`'a `/etc/appconfig` yoluna bir volume olarak mount edin.
        *   **Çözüm Yaklaşımı:** Pod manifestosunda `volumes` altında `configMap` ve `containers.volumeMounts` tanımlanır.
        *   **Efektif Çözüm:** Pod YAML'ı düzenlenir:
            ```yaml
            # ...
            spec:
              containers:
              - name: main-container
                # ...
                volumeMounts:
                - name: config-volume
                  mountPath: /etc/appconfig
              volumes:
              - name: config-volume
                configMap:
                  name: app-config
            # ...
            ```
            Kontrol: `kubectl exec my-pod -n <namespace> -- ls /etc/appconfig`
    *   **Use Case 4: Değişmez (Immutable) ConfigMap Oluşturma**
        *   **Senaryo:** `stable-config` adında, `version=1.0` verisiyle değişmez bir ConfigMap oluşturun.
        *   **Çözüm Yaklaşımı:** ConfigMap manifestosunda `immutable: true` ayarlanır.
        *   **Efektif Çözüm:**
            ```yaml
            apiVersion: v1
            kind: ConfigMap
            metadata:
              name: stable-config
            data:
              version: "1.0"
            immutable: true
            ```
            `kubectl apply -f configmap.yaml -n <namespace>`

*   **Secrets** (ConfigMap'lere benzer use case'ler, sadece `secretRef` ve `secretKeyRef` kullanılır ve `kubectl create secret generic` veya `docker-registry` kullanılır)
    *   **Use Case 1: Ortam Değişkeni Olarak Kullanma**
        *   **Senaryo:** `db-credentials` adında bir Secret oluşturun (`username=admin`, `password=securepassword`). Bu Secret'taki `password` anahtarını `DB_PASSWORD` ortam değişkeni olarak bir Pod'a atayın.
        *   **Çözüm Yaklaşımı:** Önce Secret oluşturulur, sonra Pod YAML'ında `env.valueFrom.secretKeyRef` kullanılır.
        *   **Efektif Çözüm:**
            1.  `kubectl create secret generic db-credentials --from-literal=username=admin --from-literal=password=securepassword -n <namespace>`
            2.  Pod YAML'ında:
                ```yaml
                env:
                - name: DB_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: db-credentials
                      key: password
                ```
    *   **Use Case 2: Docker Image Pull Secret Kullanma**
        *   **Senaryo:** `my-private-registry.com` adresindeki özel bir Docker registry'den imaj çekmek için `registry-secret` adında bir `imagePullSecret` oluşturun ve bunu `private-app-pod` adlı Pod'da kullanın. (Kullanıcı adı: `user`, şifre: `pass`)
        *   **Çözüm Yaklaşımı:** `kubectl create secret docker-registry` ile secret oluşturulur, sonra Pod manifestosunda `imagePullSecrets` alanına eklenir.
        *   **Efektif Çözüm:**
            1.  `kubectl create secret docker-registry registry-secret --docker-server=my-private-registry.com --docker-username=user --docker-password=pass --docker-email=user@example.com -n <namespace>`
            2.  Pod YAML'ında:
                ```yaml
                spec:
                  containers:
                  - name: private-app
                    image: my-private-registry.com/my-app:latest
                  imagePullSecrets:
                  - name: registry-secret
                ```

*   **SecurityContexts (Güvenlik Bağlamları)**
    *   **Use Case 1: Pod Seviyesinde `runAsUser` ve `fsGroup`**
        *   **Senaryo:** Bir Pod'daki tüm container'ların kullanıcı ID 1000 ve dosya sistemi grubu 2000 ile çalışmasını sağlayın.
        *   **Çözüm Yaklaşımı:** Pod manifestosunda `spec.securityContext` kullanılır.
        *   **Efektif Çözüm:** Pod YAML'ında:
            ```yaml
            spec:
              securityContext:
                runAsUser: 1000
                fsGroup: 2000
              # ... containers
            ```
    *   **Use Case 2: Container Seviyesinde `readOnlyRootFilesystem` ve `Capabilities`**
        *   **Senaryo:** Bir Pod içindeki `secure-container` adlı container'ın root dosya sisteminin salt okunur olmasını ve sadece `NET_BIND_SERVICE` yeteneğine sahip olmasını, diğer tüm yeteneklerin düşürülmesini sağlayın.
        *   **Çözüm Yaklaşımı:** Container tanımında `securityContext` kullanılır.
        *   **Efektif Çözüm:** Pod YAML'ında, ilgili container altında:
            ```yaml
            securityContext:
              readOnlyRootFilesystem: true
              capabilities:
                drop:
                - ALL
                add:
                - NET_BIND_SERVICE
            ```

*   **ServiceAccounts**
    *   **Use Case 1: Pod'a Özel ServiceAccount Atama**
        *   **Senaryo:** `app-runner` adında bir ServiceAccount oluşturun. `my-app-pod` adlı Pod'un bu ServiceAccount'ı kullanmasını sağlayın.
        *   **Çözüm Yaklaşımı:** Önce ServiceAccount oluşturulur, sonra Pod manifestosunda `serviceAccountName` belirtilir.
        *   **Efektif Çözüm:**
            1.  `kubectl create serviceaccount app-runner -n <namespace>`
            2.  Pod YAML'ında: `spec: serviceAccountName: app-runner`
    *   **Use Case 2: Role ve RoleBinding ile Yetkilendirme**
        *   **Senaryo:** `config-reader-sa` adlı ServiceAccount'a, `<namespace>` içindeki ConfigMap'leri okuma (`get`, `list`) yetkisi verin.
        *   **Çözüm Yaklaşımı:** Bir Role (ConfigMap okuma kurallarıyla) ve bu Role'ü ServiceAccount'a bağlayan bir RoleBinding oluşturulur.
        *   **Efektif Çözüm:**
            1.  `kubectl create role configmap-reader --verb=get,list,watch --resource=configmaps -n <namespace>`
            2.  `kubectl create rolebinding config-reader-binding --role=configmap-reader --serviceaccount=<namespace>:config-reader-sa -n <namespace>`
            (ServiceAccount'ın önceden oluşturulduğunu varsayıyoruz: `kubectl create sa config-reader-sa -n <namespace>`)

*   **Resource Requirements (Kaynak Gereksinimleri)**
    *   **Use Case:** Bir container için CPU isteğini "200m" (0.2 CPU), CPU limitini "500m", bellek isteğini "128Mi", bellek limitini "256Mi" olarak ayarlayın.
    *   **Çözüm Yaklaşımı:** Container tanımında `resources.requests` ve `resources.limits` alanları kullanılır.
    *   **Efektif Çözüm:** Pod YAML'ında, ilgili container altında:
        ```yaml
        resources:
          requests:
            memory: "128Mi"
            cpu: "200m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        ```
        `kubectl set resources deployment <name> -c <container> --limits=cpu=500m,memory=256Mi --requests=cpu=200m,memory=128Mi` komutu da kullanılabilir (Deployment için).

*   **Commands and Arguments (Komutlar ve Argümanlar)**
    *   **Use Case:** Bir Pod'daki container'ın varsayılan Docker imajı ENTRYPOINT'ini `/app/run.sh` komutuyla ve argümanlarını `["--mode=production", "--port=8080"]` ile override edin.
    *   **Çözüm Yaklaşımı:** Container tanımında `command` ve `args` alanları kullanılır.
    *   **Efektif Çözüm:** Pod YAML'ında, ilgili container altında:
        ```yaml
        command: ["/app/run.sh"]
        args: ["--mode=production", "--port=8080"]
        ```

*   **Taints and Tolerations (Lekeler ve Toleranslar)**
    *   **Use Case:** `node01` adlı node'a `app=critical:NoSchedule` taint'i ekleyin. Bu node'da sadece bu taint'e toleransı olan bir Pod (`critical-pod`) çalıştırın.
    *   **Çözüm Yaklaşımı:** Node'a taint eklenir, Pod manifestosuna tolerans eklenir.
    *   **Efektif Çözüm:**
        1.  `kubectl taint node node01 app=critical:NoSchedule`
        2.  `critical-pod` YAML'ında:
            ```yaml
            spec:
              tolerations:
              - key: "app"
                operator: "Equal"
                value: "critical"
                effect: "NoSchedule"
              # ... containers
            ```

*   **Node Selectors (Düğüm Seçiciler)**
    *   **Use Case:** Bir Pod'un sadece `disktype=ssd` etiketine sahip node'larda çalışmasını sağlayın.
    *   **Çözüm Yaklaşımı:** Pod manifestosunda `spec.nodeSelector` kullanılır.
    *   **Efektif Çözüm:** Pod YAML'ında:
        ```yaml
        spec:
          nodeSelector:
            disktype: ssd
          # ... containers
        ```

*   **Node Affinity (Düğüm Eğilimi)**
    *   **Use Case:** Bir Pod'un `zone=us-west-1` etiketli node'larda çalışmasını zorunlu kılın (`requiredDuringSchedulingIgnoredDuringExecution`).
    *   **Çözüm Yaklaşımı:** Pod manifestosunda `spec.affinity.nodeAffinity` kullanılır.
    *   **Efektif Çözüm:** Pod YAML'ında:
        ```yaml
        spec:
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                - matchExpressions:
                  - key: zone
                    operator: In
                    values:
                    - us-west-1
          # ... containers
        ```

*   **Pod Affinity and Anti-Affinity (Pod Eğilimi ve Karşı Eğilimi)**
    *   **Use Case (Pod Affinity):** `frontend-pod`'un, `service=backend` etiketli Pod'larla aynı node'da çalışmasını tercih edin.
    *   **Çözüm Yaklaşımı:** `frontend-pod` YAML'ında `spec.affinity.podAffinity` kullanılır.
    *   **Efektif Çözüm:** `frontend-pod` YAML'ında:
        ```yaml
        spec:
          affinity:
            podAffinity:
              preferredDuringSchedulingIgnoredDuringExecution:
              - weight: 100
                podAffinityTerm:
                  labelSelector:
                    matchExpressions:
                    - key: service
                      operator: In
                      values:
                      - backend
                  topologyKey: "kubernetes.io/hostname"
          # ... containers
        ```

### MultiContainer Pods (Çoklu Konteyner Pod'ları)

*   **Sidecar Pattern**
    *   **Use Case:** Ana uygulama container'ının (`main-app`) loglarını `/var/log/app.log` dosyasına yazdığını varsayalım. Bu logları okuyup standart çıktıya yazan bir `log-agent` sidecar container'ı ekleyin. İki container arasında log dosyası için bir `emptyDir` volume kullanın.
    *   **Çözüm Yaklaşımı:** Pod manifestosuna ikinci bir container (sidecar) ve paylaşımlı `emptyDir` volume eklenir.
    *   **Efektif Çözüm:** Pod YAML'ında:
        ```yaml
        spec:
          containers:
          - name: main-app
            image: #...
            volumeMounts:
            - name: shared-logs
              mountPath: /var/log
          - name: log-agent
            image: busybox
            command: ["sh", "-c", "tail -f /mnt/logs/app.log"]
            volumeMounts:
            - name: shared-logs
              mountPath: /mnt/logs
              readOnly: true
          volumes:
          - name: shared-logs
            emptyDir: {}
        ```
*   **Init Containers**
    *   **Use Case:** Ana uygulama container'ı başlamadan önce, `google.com` adresine ping atarak ağ bağlantısını kontrol eden bir init container (`network-check`) oluşturun.
    *   **Çözüm Yaklaşımı:** Pod manifestosunda `spec.initContainers` kullanılır.
    *   **Efektif Çözüm:** Pod YAML'ında:
        ```yaml
        spec:
          initContainers:
          - name: network-check
            image: busybox
            command: ['sh', '-c', 'until ping -c1 google.com; do echo "Waiting for network..."; sleep 2; done;']
          containers:
          # ... ana container(lar)
        ```

### Observability (Gözlemlenebilirlik)

*   **Readiness Probes**
    *   **Use Case:** Bir web sunucusu container'ının `/ready` endpoint'ine (port 8080) HTTP GET isteği göndererek hazır olup olmadığını kontrol eden bir readiness probe tanımlayın. Başlangıçta 5 saniye gecikme, her 10 saniyede bir kontrol.
    *   **Çözüm Yaklaşımı:** Container tanımında `readinessProbe` kullanılır.
    *   **Efektif Çözüm:** Pod YAML'ında, ilgili container altında:
        ```yaml
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        ```
*   **Liveness Probes**
    *   **Use Case:** Bir uygulamanın 9000 portunda TCP bağlantısı kabul edip etmediğini kontrol eden bir liveness probe tanımlayın. Eğer 3 başarısız denemeden sonra hala yanıt vermiyorsa container'ı yeniden başlatın.
    *   **Çözüm Yaklaşımı:** Container tanımında `livenessProbe` kullanılır.
    *   **Efektif Çözüm:** Pod YAML'ında, ilgili container altında:
        ```yaml
        livenessProbe:
          tcpSocket:
            port: 9000
          initialDelaySeconds: 15
          periodSeconds: 20
          failureThreshold: 3
        ```
*   **Startup Probes**
    *   **Use Case:** Yavaş başlayan bir uygulama için, ilk 2 dakika boyunca her 5 saniyede bir `/startup` endpoint'ini (port 80) kontrol eden bir startup probe tanımlayın. Bu probe başarılı olana kadar liveness/readiness probe'ları devreye girmesin.
    *   **Çözüm Yaklaşımı:** Container tanımında `startupProbe` kullanılır.
    *   **Efektif Çözüm:** Pod YAML'ında, ilgili container altında:
        ```yaml
        startupProbe:
          httpGet:
            path: /startup
            port: 80
          failureThreshold: 24 # (2 dakika / 5 saniye = 24)
          periodSeconds: 5
        ```
*   **Container Logging (`kubectl logs`)**
    *   **Use Case 1:** `my-multi-container-pod` adlı Pod'un `sidecar-container` adlı container'ının loglarını görüntüleyin.
    *   **Çözüm Yaklaşımı:** `kubectl logs <pod-name> -c <container-name>`
    *   **Efektif Çözüm:** `kubectl logs my-multi-container-pod -c sidecar-container -n <namespace>`
    *   **Use Case 2:** Bir Pod'un önceki (crash olmuş) container'ının loglarını görüntüleyin.
    *   **Çözüm Yaklaşımı:** `kubectl logs <pod-name> --previous`
    *   **Efektif Çözüm:** `kubectl logs my-crashed-pod --previous -n <namespace>`
*   **Monitor and Debug Applications**
    *   **Use Case 1 (`kubectl describe`):** `broken-pod` adlı bir Pod neden `Pending` durumda? Veya bir Deployment neden istediği sayıda Pod oluşturamıyor?
    *   **Çözüm Yaklaşımı:** `kubectl describe pod <pod-name>` veya `kubectl describe deployment <deployment-name>`. Özellikle `Events` bölümüne bakın.
    *   **Efektif Çözüm:** `kubectl describe pod broken-pod -n <namespace>`
    *   **Use Case 2 (`kubectl exec`):** `web-server-pod` adlı Pod'un içindeki `nginx-container`'da `/etc/nginx/nginx.conf` dosyasının içeriğini görüntüleyin.
    *   **Çözüm Yaklaşımı:** `kubectl exec <pod-name> -c <container-name> -- <command>`
    *   **Efektif Çözüm:** `kubectl exec web-server-pod -c nginx-container -n <namespace> -- cat /etc/nginx/nginx.conf`
    *   **Use Case 3 (`kubectl port-forward`):** `my-api-pod` adlı Pod'un 8080 portunda çalışan servisine lokal makinenizin 9090 portundan erişin.
    *   **Çözüm Yaklaşımı:** `kubectl port-forward pod/<pod-name> <local-port>:<pod-port>`
    *   **Efektif Çözüm:** `kubectl port-forward pod/my-api-pod 9090:8080 -n <namespace>`
    *   **Use Case 4 (`kubectl top`):** `<namespace>` içindeki Pod'ların CPU ve bellek kullanımını sıralı olarak görüntüleyin. (Metrics Server kurulu olmalı)
    *   **Çözüm Yaklaşımı:** `kubectl top pod -n <namespace> --sort-by=cpu`
    *   **Efektif Çözüm:** `kubectl top pod -n <namespace> --sort-by=memory`

### POD Design (POD Tasarımı)

*   **Labels, Selectors and Annotations**
    *   **Use Case 1 (Label Ekleme):** Mevcut `my-app-pod` adlı Pod'a `env=production` ve `tier=frontend` etiketlerini ekleyin.
    *   **Çözüm Yaklaşımı:** `kubectl label pod <pod-name> <label-key>=<label-value>`
    *   **Efektif Çözüm:** `kubectl label pod my-app-pod env=production tier=frontend -n <namespace> [--overwrite]`
    *   **Use Case 2 (Selector ile Listeleme):** `env=production` VE `tier=frontend` etiketlerine sahip tüm Pod'ları listeleyin.
    *   **Çözüm Yaklaşımı:** `kubectl get pods -l <label-key>=<label-value>,<another-key>=<another-value>`
    *   **Efektif Çözüm:** `kubectl get pods -l env=production,tier=frontend -n <namespace>`
    *   **Use Case 3 (Annotation Ekleme):** `my-deployment` adlı Deployment'a `description="Main web application"` şeklinde bir ek açıklama ekleyin.
    *   **Çözüm Yaklaşımı:** `kubectl annotate deployment <deployment-name> <annotation-key>=<annotation-value>`
    *   **Efektif Çözüm:** `kubectl annotate deployment my-deployment description="Main web application" -n <namespace> [--overwrite]`
*   **Deployments**
    *   **Use Case 1 (Deployment Oluşturma ve Scale Etme):** `nginx` imajını kullanan, 3 replikalı `web-frontend` adında bir Deployment oluşturun.
    *   **Çözüm Yaklaşımı:** `kubectl create deployment` ve `kubectl scale deployment`.
    *   **Efektif Çözüm:**
        1.  `kubectl create deployment web-frontend --image=nginx -n <namespace>`
        2.  `kubectl scale deployment web-frontend --replicas=3 -n <namespace>`
    *   **Use Case 2 (Rolling Update ve Strateji):** `web-frontend` Deployment'ının imajını `nginx:1.25.0` olarak güncelleyin. Güncelleme sırasında en fazla 1 Pod'un kullanılamaz (`maxUnavailable`) ve istenen replika sayısının en fazla %25 üzerinde Pod (`maxSurge`) olmasını sağlayın.
    *   **Çözüm Yaklaşımı:** `kubectl set image` ve Deployment YAML'ında `strategy.rollingUpdate` ayarları.
    *   **Efektif Çözüm:**
        1.  Deployment YAML'ını düzenle (`kubectl edit deployment web-frontend -n <namespace>`):
            ```yaml
            spec:
              strategy:
                type: RollingUpdate
                rollingUpdate:
                  maxUnavailable: 1
                  maxSurge: 25%
            ```
        2.  `kubectl set image deployment/web-frontend nginx=nginx:1.25.0 -n <namespace>`
    *   **Use Case 3 (Rollback):** `web-frontend` Deployment'ını bir önceki çalışan sürüme geri döndürün.
    *   **Çözüm Yaklaşımı:** `kubectl rollout undo deployment`
    *   **Efektif Çözüm:**
        1.  Geçmişi gör: `kubectl rollout history deployment/web-frontend -n <namespace>`
        2.  Geri al: `kubectl rollout undo deployment/web-frontend -n <namespace>` (Belirli bir revizyona dönmek için `--to-revision=<number>`)
*   **Jobs and CronJobs**
    *   **Use Case 1 (Job Oluşturma):** `perl` imajını kullanarak "Hello Kubernetes Batch Job" çıktısı veren bir Job (`hello-job`) oluşturun. Job'un 5 kez başarıyla tamamlanmasını (`completions: 5`) ve aynı anda en fazla 2 Pod (`parallelism: 2`) çalıştırmasını sağlayın.
    *   **Çözüm Yaklaşımı:** Job manifestosu oluşturulur.
    *   **Efektif Çözüm:** `hello-job.yaml`:
        ```yaml
        apiVersion: batch/v1
        kind: Job
        metadata:
          name: hello-job
        spec:
          completions: 5
          parallelism: 2
          template:
            spec:
              containers:
              - name: hello-container
                image: perl
                command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(20)"] # Örnek bir komut
              restartPolicy: OnFailure # veya Never
        ```
        `kubectl apply -f hello-job.yaml -n <namespace>`
    *   **Use Case 2 (CronJob Oluşturma):** Her dakika `busybox` imajını kullanarak "Date: <tarih>" çıktısı veren bir CronJob (`minute-cronjob`) oluşturun.
    *   **Çözüm Yaklaşımı:** CronJob manifestosu oluşturulur.
    *   **Efektif Çözüm:** `kubectl create cronjob minute-cronjob --image=busybox --schedule="*/1 * * * *" -- /bin/sh -c 'date; echo "Hello from the Kubernetes CronJob"' -n <namespace> --dry-run=client -o yaml > cronjob.yaml` ile başlayıp düzenlenebilir veya doğrudan YAML:
        ```yaml
        apiVersion: batch/v1
        kind: CronJob
        metadata:
          name: minute-cronjob
        spec:
          schedule: "*/1 * * * *"
          jobTemplate:
            spec:
              template:
                spec:
                  containers:
                  - name: cron-container
                    image: busybox
                    command:
                    - /bin/sh
                    - -c
                    - date; echo "Hello from the Kubernetes CronJob"
                  restartPolicy: OnFailure
        ```
        `kubectl apply -f cronjob.yaml -n <namespace>`

### Services and Networking (Servisler ve Ağ İletişimi)

*   **Service Tipleri:**
    *   **Use Case 1 (ClusterIP Service Oluşturma):** `app=my-web-app` etiketli Pod'ları hedefleyen, `my-web-service` adında, 80 portunu Pod'ların 8080 portuna yönlendiren bir ClusterIP Service oluşturun.
    *   **Çözüm Yaklaşımı:** `kubectl expose deployment` (eğer bir Deployment varsa) veya Service YAML'ı.
    *   **Efektif Çözüm:** (Eğer `my-web-app-deployment` adlı bir deployment varsa)
        `kubectl expose deployment my-web-app-deployment --port=80 --target-port=8080 --name=my-web-service -n <namespace>`
        (Type varsayılan olarak ClusterIP'dir)
    *   **Use Case 2 (NodePort Service Oluşturma):** `my-web-service` adlı servisi, node'ların 30080 portu üzerinden dışarıdan erişilebilir hale getirin.
    *   **Çözüm Yaklaşımı:** Service manifestosunda `type: NodePort` ve `nodePort` belirtilir veya `kubectl edit service`.
    *   **Efektif Çözüm:** `kubectl edit service my-web-service -n <namespace>` ve `type: NodePort`, `ports[0].nodePort: 30080` ekleyin. Veya `kubectl expose ... --type=NodePort --port=80 --target-port=8080 --node-port=30080` (expose ile nodePort belirtmek için YAML gerekebilir veya patch). En kolayı `kubectl expose deployment my-web-app-deployment --port=80 --target-port=8080 --name=my-nodeport-service --type=NodePort -n <namespace>`. Sonra `kubectl patch svc my-nodeport-service -p '{"spec":{"ports":[{"port":80,"nodePort":30080}]}}' -n <namespace>` (port numarasını koruyarak nodePort eklemek için).
*   **Ingress Resource ve Ingress Controller**
    *   **Use Case:** `example.com/app` yoluna gelen istekleri `app-service` (port 80) adlı servise, `example.com/api` yoluna gelen istekleri ise `api-service` (port 8080) adlı servise yönlendiren bir Ingress (`my-ingress`) kaynağı oluşturun.
    *   **Çözüm Yaklaşımı:** Ingress manifestosu oluşturulur. (Ingress controller'ın kurulu olduğu varsayılır).
    *   **Efektif Çözüm:** `my-ingress.yaml`:
        ```yaml
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        metadata:
          name: my-ingress
          # annotations:
          #   nginx.ingress.kubernetes.io/rewrite-target: / # Eğer path rewrite gerekiyorsa
        spec:
          rules:
          - host: example.com # Opsiyonel, host belirtmezseniz tüm hostlar için geçerli olur
            http:
              paths:
              - path: /app
                pathType: Prefix
                backend:
                  service:
                    name: app-service
                    port:
                      number: 80
              - path: /api
                pathType: Prefix
                backend:
                  service:
                    name: api-service
                    port:
                      number: 8080
        ```
        `kubectl apply -f my-ingress.yaml -n <namespace>`
*   **Network Policies (Ağ Politikaları)**
    *   **Use Case 1 (Default Deny Ingress):** `<namespace>` içindeki tüm Pod'lara gelen trafiği varsayılan olarak engelleyin.
    *   **Çözüm Yaklaşımı:** `podSelector: {}` ile tüm Pod'ları hedefleyen ve boş `ingress: []` kuralı olan bir NetworkPolicy oluşturulur.
    *   **Efektif Çözüm:** `default-deny-ingress.yaml`:
        ```yaml
        apiVersion: networking.k8s.io/v1
        kind: NetworkPolicy
        metadata:
          name: default-deny-ingress
        spec:
          podSelector: {} # Namespace'deki tüm podları seçer
          policyTypes:
          - Ingress
          # ingress: [] # Boş ingress kuralı tüm gelen trafiği engeller (bu satır olmasa da olur, policyTypes'da Ingress varsa ve kural yoksa deny)
        ```
        `kubectl apply -f default-deny-ingress.yaml -n <namespace>`
    *   **Use Case 2 (Belirli Pod'lardan İzin):** `role=backend` etiketli Pod'lara sadece `role=frontend` etiketli Pod'lardan 6379 TCP portuna gelen trafiğe izin verin.
    *   **Çözüm Yaklaşımı:** `podSelector` ile `role=backend`'i hedefleyen, `ingress` kuralında `from` ve `ports` belirten bir NetworkPolicy.
    *   **Efektif Çözüm:** `backend-policy.yaml`:
        ```yaml
        apiVersion: networking.k8s.io/v1
        kind: NetworkPolicy
        metadata:
          name: backend-access-policy
        spec:
          podSelector:
            matchLabels:
              role: backend
          policyTypes:
          - Ingress
          ingress:
          - from:
            - podSelector:
                matchLabels:
                  role: frontend
            ports:
            - protocol: TCP
              port: 6379
        ```
        `kubectl apply -f backend-policy.yaml -n <namespace>`

### State Persistence (Durum Kalıcılığı)

*   **Volumes (Birimler)**
    *   **Use Case (`emptyDir`):** Bir Pod'daki iki container arasında geçici veri paylaşımı için `shared-data` adında bir `emptyDir` volume kullanın. Bir container `/producer` yoluna yazsın, diğeri `/consumer` yolundan okusun.
    *   **Çözüm Yaklaşımı:** Pod YAML'ında `volumes` ve `volumeMounts` tanımlanır.
    *   **Efektif Çözüm:** Pod YAML'ında:
        ```yaml
        spec:
          containers:
          - name: producer-container
            image: busybox
            command: ["/bin/sh", "-c", "echo 'Hello from producer' > /producer/data.txt && sleep 3600"]
            volumeMounts:
            - name: shared-data
              mountPath: /producer
          - name: consumer-container
            image: busybox
            command: ["/bin/sh", "-c", "sleep 5 && cat /consumer/data.txt && sleep 3600"]
            volumeMounts:
            - name: shared-data
              mountPath: /consumer
          volumes:
          - name: shared-data
            emptyDir: {}
        ```
*   **Persistent Volume Claims (PVCs)**
    *   **Use Case 1 (PVC Oluşturma):** `my-app-storage` adında, 1Gi boyutunda, `ReadWriteOnce` erişim modu isteyen ve (eğer varsa) `standard` StorageClass'ını kullanan bir PVC oluşturun.
    *   **Çözüm Yaklaşımı:** PVC manifestosu oluşturulur.
    *   **Efektif Çözüm:** `my-pvc.yaml`:
        ```yaml
        apiVersion: v1
        kind: PersistentVolumeClaim
        metadata:
          name: my-app-storage
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 1Gi
          storageClassName: standard # Opsiyonel, varsayılan SC kullanılırsa gerekmez
        ```
        `kubectl apply -f my-pvc.yaml -n <namespace>`
    *   **Use Case 2 (Pod'a PVC Mount Etme):** `my-app-pod` adlı bir Pod'a, `my-app-storage` PVC'sini `/data` yoluna mount edin.
    *   **Çözüm Yaklaşımı:** Pod YAML'ında `volumes` altında `persistentVolumeClaim` ve `volumeMounts` tanımlanır.
    *   **Efektif Çözüm:** Pod YAML'ında:
        ```yaml
        spec:
          containers:
          - name: app-container
            image: #...
            volumeMounts:
            - name: app-data
              mountPath: /data
          volumes:
          - name: app-data
            persistentVolumeClaim:
              claimName: my-app-storage
        ```
*   **StorageClasses (Depolama Sınıfları)**
    *   **Use Case:** Mevcut StorageClass'ları listeleyin ve varsayılan olanı belirleyin.
    *   **Çözüm Yaklaşımı:** `kubectl get sc`
    *   **Efektif Çözüm:** `kubectl get storageclass` (Varsayılan olanın yanında `(default)` yazar).

### Security (Güvenlik)

(RBAC, Secrets, SecurityContexts yukarıda işlendi. Burada daha çok kavramsal ve diğerleriyle birleşen senaryolar olabilir.)

*   **Minimum Yetki Prensibi (Principle of Least Privilege)**
    *   **Senaryo:** Bir ServiceAccount'a sadece belirli bir Deployment'ı (`my-deploy`) scale etme yetkisi verin.
    *   **Çözüm Yaklaşımı:** Bir Role oluşturun (`verbs: ["get", "patch", "update"]`, `resources: ["deployments/scale"]`, `resourceNames: ["my-deploy"]`). Sonra RoleBinding.
    *   **Efektif Çözüm:** Role YAML'ı:
        ```yaml
        apiVersion: rbac.authorization.k8s.io/v1
        kind: Role
        metadata:
          name: deploy-scaler
        rules:
        - apiGroups: ["apps"]
          resources: ["deployments/scale"]
          resourceNames: ["my-deploy"]
          verbs: ["get", "patch", "update"]
        - apiGroups: ["apps"] # Ölçekleme için genellikle get de gerekir
          resources: ["deployments"]
          resourceNames: ["my-deploy"]
          verbs: ["get"]
        ```
        Sonra RoleBinding oluşturulur.

### PodDisruptionBudgets (PDBs)

*   **Use Case:** `app=critical-service` etiketli Pod'lardan en az %50'sinin her zaman çalışır durumda olmasını garanti eden bir PDB (`critical-pdb`) oluşturun.
*   **Çözüm Yaklaşımı:** PDB manifestosu oluşturulur.
*   **Efektif Çözüm:** `pdb.yaml`:
    ```yaml
    apiVersion: policy/v1
    kind: PodDisruptionBudget
    metadata:
      name: critical-pdb
    spec:
      minAvailable: 50% # veya maxUnavailable: 2 (eğer toplam 4 pod varsa)
      selector:
        matchLabels:
          app: critical-service
    ```
    `kubectl apply -f pdb.yaml -n <namespace>`

### ResourceQuotas ve LimitRanges

*   **ResourceQuotas**
    *   **Use Case:** `<dev-namespace>` için toplamda en fazla 5 Pod, 2 CPU (request) ve 4Gi bellek (request) kotası tanımlayın.
    *   **Çözüm Yaklaşımı:** ResourceQuota manifestosu oluşturulur.
    *   **Efektif Çözüm:** `quota.yaml`:
        ```yaml
        apiVersion: v1
        kind: ResourceQuota
        metadata:
          name: dev-ns-quota
          namespace: <dev-namespace> # Namespace burada belirtilmeli
        spec:
          hard:
            pods: "5"
            requests.cpu: "2"
            requests.memory: "4Gi"
            # limits.cpu: "4" # Limitler de eklenebilir
            # limits.memory: "8Gi"
        ```
        `kubectl apply -f quota.yaml` (Namespace önceden oluşturulmuş olmalı)
*   **LimitRanges**
    *   **Use Case:** `<test-namespace>` içindeki container'lar için varsayılan CPU isteğini 100m, bellek isteğini 64Mi; maksimum CPU limitini 500m, maksimum bellek limitini 512Mi olarak ayarlayın.
    *   **Çözüm Yaklaşımı:** LimitRange manifestosu oluşturulur.
    *   **Efektif Çözüm:** `limitrange.yaml`:
        ```yaml
        apiVersion: v1
        kind: LimitRange
        metadata:
          name: test-ns-limits
          namespace: <test-namespace> # Namespace burada belirtilmeli
        spec:
          limits:
          - type: Container
            defaultRequest:
              cpu: "100m"
              memory: "64Mi"
            default: # Varsayılan limitler (eğer request belirtilmişse limit de bu olur)
              cpu: "200m"
              memory: "128Mi"
            max:
              cpu: "500m"
              memory: "512Mi"
        ```
        `kubectl apply -f limitrange.yaml`
