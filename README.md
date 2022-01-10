## Bağlantılar

- [prometheus.yaml Ayar dosyasında olabilecek job tanımlarına dair örnekler](https://github.com/prometheus/prometheus/blob/0a8d28ea932ed18e000c2f091200a46d2b62bac4/config/testdata/conf.good.yml)



## nginx Exporter
Prometheus sunucusu ve metrik üreten bir nginx sunucusu ayakalnıyor.
Nginx sunucusu metrics isimli dosyayı 80 portundan sanki bir uygulamanın metriklerini sunuyor gibi yayımlıyor.
Nginx 80 dahili portunda çalışıyor ve hostun 80 portundan erişilebiliyor.
http://localhost:80/metrics

Nginx ayar dosyasında başarılı işlemleri ayrı dosyalara günlüklüyoruz çünkü bu günlükleri promtail ile çekeceğiz ve lokiye göndereceğiz.

```nginx
  access_log /var/log/nginx/metrik-ornegi.access.log;
  error_log  /var/log/nginx/metrik-ornegi.error.log  debug;
```

## PROMTAIL
![image](https://user-images.githubusercontent.com/261946/148707175-568955e7-b995-4b40-81a9-bb93e002330c.png)

Promtail'in amacı log okumak ve lokiye bu günlük kayıtlarını itmek.
Çalıştığını  [http://localhost:9080](http://localhost:3100)
Nginx konteynerinin `/var/log/nginx` dizinini bir `nginx-log-data` isminde volume yaratarak bu dizine bağlıyoruz.
Bu volume, host üstünde ortak nokta olacak Nginx ile Promtail konteynerleri arasında.
Promtail'in scrape ayarlarına, kendi üstünde `/var/log/nginx` dizinindeki uzantısı `log` olan dosyaları tara ve loki sunucusuna (`cloki` konteynerine) gönder diyoruz:

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://cloki:3100/loki/api/v1/push

scrape_configs:
- job_name: ncin
  static_configs:
  - targets:
      - localhost
    labels:
      job: ncinlog
      __path__: /var/log/nginx/*log
``` 

## LOKI
![image](https://user-images.githubusercontent.com/261946/148707231-728e923c-1559-4a29-9077-bba75b13f430.png)

Görevi gelen log bilgilerini kendi üstünde saklamak ve Grafana sorduğunda ona sunmak.
Loki varsayılan ayarları ile veya bu projedeki `loki-config.yaml` dosyasıyla başlatılabilir.
Loki çalıştığı zaman, 3100 portunda metriklerini gösterecektir [http://localhost:3100](http://localhost:3100).

## PROMETHEUS
![image](https://user-images.githubusercontent.com/261946/148707225-456079fe-5938-4379-b15c-f0f94351b994.png)

Prometheus varsayılan olarak `/metrics` adresine gider ve eğer farklı adrese gitmesi istenirse `metric_path` alanı job içinde tanımlanmalıdır:

```yaml
- job_name: "ornek job"
  metrics_path: /sunucu-adresine/ek-olacak/metrik-adresi
  static_configs:
    targets: ["sunucu-adresi.com", "192.168.1.19", ...]
```

Prometheus konteyneri içeride 8090 portunda çalışırken dışarıda 90 portundan erişilebilir. 
http://localhost:90

## Docker Komutları
Tüm konteynerleri >
Yansıları derlemek için:

```shell
docker-compose -f .\docker-compose.yaml build
```

Çalıştırmak için
```shell
docker-compose -f .\docker-compose.yaml up
```

Durdurmak için:

```shell
docker-compose -f .\docker-compose.yaml down
```
## GRAFANA
![image](https://user-images.githubusercontent.com/261946/148707250-99b5db6d-c17d-439e-82d7-7f6406360ee6.png)

Sadece Grafayı ayaklandırmak için:
```shell
 docker run -d \
     -p 3000:3000 \
     --name=grafana \
     -e "GF_INSTALL_PLUGINS=grafana-clock-panel,grafana-simple-json-datasource" \
     grafana/grafana
```

Grafana'yı başlatmak için http://localhost:3000 adresine (kullanıcı adı ve şifresi "admin" ile) girilir ve Data Source kısmına Prometheus eklenerek Prometheus'un konteynerler arasında erişilebileceği URL adresi (http://cprom:9090) yazılarak "Save & Test" düğmesine basılarak eklenir.

