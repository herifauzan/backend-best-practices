Backend development best practices
==================================

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [N Commandments](#n-commandments)
- [General points on guidelines](#general-points-on-guidelines)
- [Development environment setup in README.md](#development-environment-setup-in-readmemd)
- [Data persistence](#data-persistence)
  - [General considerations](#general-considerations)
  - [SaaS, cloud-hosted or self-hosted?](#saas-cloud-hosted-or-self-hosted)
  - [Persistence solutions](#persistence-solutions)
    - [RDBMS](#rdbms)
    - [NoSQL](#nosql)
      - [Document storage](#document-storage)
      - [Key-value store](#key-value-store)
      - [Graph database](#graph-database)
- [Environments](#environments)
  - [Local development environment](#local-development-environment)
  - [Continuous integration environment](#continuous-integration-environment)
  - [Testing environment](#testing-environment)
  - [Staging environment](#staging-environment)
  - [Production environment](#production-environment)
- [Bill of Materials](#bill-of-materials)
- [Security](#security)
  - [Docker](#docker)
  - [Credentials](#credentials)
  - [Secrets](#secrets)
  - [Login Throttling](#login-throttling)
  - [User Password Storage](#user-password-storage)
  - [Audit Log](#audit-log)
  - [Suspicious Action Throttling and/or blocking](#suspicious-action-throttling-andor-blocking)
  - [Anonymized Data](#anonymized-data)
  - [Temporary file storage](#temporary-file-storage)
  - [Dedicated vs Shared server environment](#dedicated-vs-shared-server-environment)
- [Application monitoring](#application-monitoring)
  - [Status page](#status-page)
  - [Status page format](#status-page-format)
    - [Plain format](#plain-format)
    - [JSON format](#json-format)
  - [HTTP status codes](#http-status-codes)
  - [Load balancer health checks](#load-balancer-health-checks)
  - [Access control](#access-control)
- [Checklists](#checklists)
  - [Responsibility checklist](#responsibility-checklist)
  - [Release checklist](#release-checklist)
- [General questions to consider](#general-questions-to-consider)
- [Generally proven useful tools](#generally-proven-useful-tools)
- [License](#license)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


# N Commandments

1. README.md pada root folder repository sebagai dokumen
2. Single command untuk menjalankan aplikasi
3. Single command untuk deploy aplikasi
4. Hasil build harus repeatable and re-creatable
5. Buat artifacts bundle yang bernama ["Bill of Materials"](#bill-of-materials)
6. Gunakan [UTC sebagai timezone](http://yellerapp.com/posts/2015-01-12-the-worst-server-setup-you-can-make.html) dimanapun

# General points on guidelines

Dokumen ini tidak membatasi atau bergantung pada stack teknologi atau framework tertentu. Masalah yang berbeda memiliki solusi yang berbeda, sehingga guideline ini valid untuk berbagai backend architectures.

# Development environment setup in README.md

Dokumentasikan seluruh bagian dari environment development/server yang dibutuhkan. Usahakan untuk menggunakan setup dan versio yang sama pada setiap environment, dimulai dari laptop developer, dan berakhir pada environment pada production. Ini termasuk database, server aplikasi,server proxy (nginx, apache, dll), versi SDK, gems/libraries/modules. 

Buatlah proses setup menjadi otomatis sebisa mungkin. Sebagai contoh, [Docker Compose](https://docs.docker.com/compose/) bisa digunakan pada lingkungan production dan development untuk melakukan setup seluruh bagian environment, dimana [Dockerfiles](https://docs.docker.com/articles/dockerfile_best-practices/) melakukan fetching seluruh bagian dari software, dan berisi script yang diperlukan untuk men-setup environment serta seluruh bagiannya. Pertimbangkan untuk menggunakan copy-an installer yang telah diarsipkan untuk berjaga-jaga jika paket installer tersebut sudah tidak tersedia lagi. Tindakan paling minimum yang bisa dilakukan adalah dengan menyimpan hasil SHA-1 checksum dari paket tersebut, dan pastikan bahwa paket yang diinstall memiliki hasil checksum yang sama dengan hasil checksum yang digunakan.

Pertimbangkan untuk menyimpan semua bagian relevan dari development environment dan seluruh dependency-nya pada persistent storage. Jika environment dapat di-build menggunakan docker, salah satu cara untuk mendapatkan ini adalah dengan menggunakan [docker export](http://docs.docker.com/reference/commandline/cli/#export).

# Data persistence

## General considerations

Independent of the persistence solution your project uses, there are general considerations that you should follow:

* Memiliki backup yang sudah terverifikasi dapat berjalan dengan normal
* Memiliki scripts atau tooling untuk meng-copy persistent data dari satu environment ke environment yang lain
* Memiliki rencana untuk meluncurkan updates pada persistence solution (e.g. database server security updates)
* Memiliki rencana untuk scaling up pada persistence solution
* Memiliki rencana atau tooling untuk me-manage schema changes
* Memiliki monitoring untuk memverifikasi healt dari persistence solution

## SaaS, cloud-hosted or self-hosted?

Sebuah keputusan penting mengenai solusi apapun adalah dimana solusi itu akan dijalankan.

* SaaS -- cepat untuk dimulai, mudah untuk scale up, beberapa infrastruktur membutuhkan diperbolehkannya akses dari banyak tempat
* Self-hosted in the cloud -- allows tuning database more than SaaS and probably cheaper at scale in terms of hosting, but more labor-intensive
* Self-hosted on own hardware -- able to tweak everything and manage physical security, but most expensive and labor intensive

## Persistence solutions

Bagian ini bertujuan untuk memberikan beberapa guidance untuk memilih tipe persistence solution. Pilihannya selalu ditujukan untuk masalah yang ingin diselesaikan dan bagaimanapun tidak ada yang namanya silver buller (Satu solusi untuk semua). 

### RDBMS

Pilihlah relational database ketika data dan integrity dari transaksi merupakan hal penting (major concern) atau ketika banyak analisis data diperlukan. Karena [ACID compliance](https://en.wikipedia.org/wiki/ACID), aggregation dan transformation functions dari RDBMS akan sangat membantu.

### NoSQL

Pilihlah NoSQL ketika tidak membutuhkan compliance pada prinsip ACID dan lebih sering untuk melakukan scale horizontally. Pilihlah sistem yang paling cocok dengan model dari data anda. Ada beberapa database NoSQL yang cocok untuk dokumen, ada yang cocok untuk gambar, ada yang cocok untuk data geografis, dan lain-lain. 

#### Document storage

Menyimpan dokumen menjadi lebih mudah ditempatkan dan dicari berdasarkan konten atau berdasarkan kesamaan pada koleksi. Tipe NoSQL ini dapat melakukannya karena database memahami format storage. Digunakan hanya untuk itu, menyimpan structured documents berjumlah banyak. Beberapa contoh:

* CouchDB
* ElasticSearch

> Catatan sejak versi 9.4, PostgreSQL juga dapat menyimpan JSON secara native.

#### Key-value store

Menyimpan values, atau terkadang kumpulan dari pasangan key-value, dapat diakses berdasarkan key. Pertimbangkan value seperti blobs, maka query tidak akan mampu digunakan oleh document store. Scalable untuk ukuran yang sangat besar. Beberapa contoh:

* Cassandra
* Redis

#### Graph database

Secara umum graph databases menyimpan node dan edge dari sebuah graf, menyediakan index-free lookups untuk melihat tetangga dari node manapun. Untuk penggunaan graf-query seperti shortest path atau diameter adalah sangat penting. Specialized graph databases juga hadir untuk penyimpanan e.g. [RDF triples](https://en.wikipedia.org/wiki/Resource_Description_Framework).

# Environments

Bagian ini mendeskripsikan environment apa saja minimal yang harus dimiliki. Mungkin akan terlihat seolah-olah banyak [tetapi ada tujuan dari masing-masingnya](http://futurice.com/blog/five-environments-you-cannot-develop-without).

- [Local development](#local-development-environment)
- [Continuous integration](#continuous-integration-environment)
- [Testing](#testing-environment)
- [Staging](#staging-environment)
- [Production](#production-environment)

## Local development environment

Ini adalah lingkungan local yang dimiliki developer. Lingkungan ini seharusnya tidak boleh mengakses environment luar manapun. Semua yang ada harus berjalan pada sistem secara lokal, dengan stubbing, mocking thirdpary services sebagaimana dibutuhkan.

## Continuous integration environment

CI adalah (antara lain) untuk menjamin bahwa software yang berhasil di-build dan lolos ketika di-test secara otomatis untuk setiap perubahan.

## Testing environment

Ini adalah shared environment dimana kode di-deploy sesering mungkin, lebih bagus jika setiap ada commit ke branch utama. Lingkungan ini dapat rusak dari waktu ke waktu, terutama ketika masa development. Ini merupakan environment yang penting dan sebisa mungkin agar sama dengan production. Segala integrasi eksternal agar diatur menggunakan staging level version dari service lainnya. 

## Staging environment

Staging harus bisa di-setup persis seratus persen sama dengan production. Tidak boleh ada perubahan di production jika tidak melewati staging terlebih dahulu. Seluruh issue aneh yang hanya terjadi di-production dapat di-debug di sini.

## Production environment

Sebuah brankas besar. Logged, monitored, cleaned up periodically, squared away and secured.

# Bill of Materials

Dokumen ini harus dimasukkan pada setiap build artifact dan harus mengandung keterangan berikut:

1. Versi berapa dari SDK dan Tools apa yang diperlukan untuk menghasilkannya
1. Dependency mana yang sudah termasuk
1. Nomor revisi global yang unik pada software (i.e. a git SHA-1 hash)
1. Environment dan variable yang digunakan ketika mem-build package
1. Daftar test atau cek yang gagal


# Security

Berhati-hatilah terhadap kemungkinan security threats dan masalah. Minimal harus familiar dengan  [OWASP Top 10 vulnerabilities](https://www.owasp.org/index.php/Category:OWASP_Top_Ten_Project), dan harus dapat memonitor vulnerabilities dari seluruh third party software yang digunakan.

Beberapa panduan security yang umum seperti:

## Docker

**Menggunakan Docker tidak akan membuat service lebih aman.** Secara umum, harus dipertimbangkan setidaknya beberapa hal ketika menggunakan Docker:

- Jangan jalankan untrusted binaries apapun di dalam Docker containers
- Buatlah unprivileged user di dalam Docker containers dan jalankan binaries menggunakan unprivileged user daripada menggunakan root kapanpun
- Secara periodik build ulang dan deploy ulang container dengan updated libraries dan dependencies
- Secara periodik update (atau rebuild) Docker hosts dengan latest security updates
- Banyak containers berjalan pada host yang sama by default akan memiliki beberapa level akses ke container lain dan hostnya itu sendiri. Amankan seluruh host dan jalankan container dengan akses capability minimum, sebagai contoh mencegah network access jika tidak dibutuhkan.

## Credentials

Jangan pernah mengirim kredensial tanpa terenkripsi melalui network public. Selalu gunakan enkripsi (such as HTTPS, SSL, etc.).

## Secrets

Jangan pernah menyimpan secrets (passwords, keys, etc.) pada source code di version control! Akan sangat mudah terlupakan jika semua secrets ada di sana dan ujungnya seluruh secret bisa diakses di banyak tempat (developer machines, development test servers, etc) yang tidak sengaja menambah resiko rahasia penting diabaikan. Juga, version control memiliki fitur berbahaya untuk melakukan overwriting file permissions, sehingga jika config file permission sudah diatur untuk diamankan, pada check out berikutnya permission akan otomatis di-overwrite menjadi default public readable.

Cara paling mungkin termudah untuk menangani hal ini adalah dengan menyimpannya pada file berbeda pada server yang memang membutuhkannya, dan dengan di-ignore oleh version control. Bisa juga dengan menyimpan `.sample` file di version control, dengan fake values untuk menjelaskan apa yang harus diubah pada konfigurasi tersebut. Pada beberapa kasus, tidak mudah memisahkan configuration file dari main configuration. Jika itu terjadi, pertimbangkan menggunakan environment variables, atau tulis config file dari template dari version control pada saat deployment.

## Login Throttling

Atur batas maksimum percobaan login yang diperbolehkan per-klien per satuan waktu. Lock akun user yang untuk waktu yang spesifik setelah gagal beberapa kali (misal. lock selama 5 menit setelah 20 kali gagal login).
Tujuan dari hal ini adalah untuk membuat brute-force attack menggunakan username dan password tidak dapat dilakukan.

## User Password Storage

> Jangan pernah menyimpan password dalam plaintext!

Jangan menyimpan password pada reversible encrypted form, kecuali sangat diperlukan oleh aplikasi/sistem. Ini adalah artikel yang bagus tentang apa yang boleh dan tidak dilakukan: https://crackstation.net/hashing-security.htm

Jika memang dibutuhkan untuk mendapatkan plaintext password dari database, ada beberapa saran sebagai berikut.

Jika password tidak akan sering di-konversi kembali ke plaintext (misal. procedure khusus dibutuhkan), simpan decryption key jauh-jauh dari aplikasi yang mengakses database secara regular.

Jika password masih butuh di-decrypt secara regular, pisahkan fungsi decrypt dari main application sebisa mungkin— misal. server terpisah menerima requests untuk men-decrypt password, tapi dipaksa harus menggunakan high level control, seperti throttling, authorization, dll.

Kapanpun memungkinkan (pada kasus yang umum mayoritas terjadi), simpan password menggunakan one-way hash yang bagus dengan sebuah random salt. SHA-1 bukanlah pilihan yang bagus untuk hashing function pada konteks ini. Hash functions yang didesign dengan password dapat dipertimbangkan sangat lambat, sehingga membuat offline brute-force attack menjadi sangat memakan waktu, akibatnya semakin tidak mungkin dilakukan. Lihat post ini untuk lebih detail: http://security.stackexchange.com/questions/211/how-to-securely-hash-passwords/31846#31846

## Audit Log

Untuk aplikasi yang meng-handle data sensitif, secara khusus jika memiliki beberapa user yang memiliki hak akses dan kontrol yang luas, langkah yang tepat untuk membuat audit loggin-storing dari sekuens aksi atau event yang terjadi pada sistem, bersama dengan sumber/event yang menjadi sumber hal tersebut (user, automation job, etc). Bisa seperti berikut, misal:

    2012-09-13 03:00:05 Job "daily_job" performed action "delete old items".
    2012-09-13 12:47:23 User "admin_user" performed action "delete item 123".
    2012-09-13 12:48:12 User "admin_user" performed action "change password of user foobar".
    2012-09-13 13:02:11 User "sneaky_user" performed action "view confidential page 567".
    ...

Mungkin log hanyalah berupa text sederhana yang tersimpan di databases. Setidaknya 3 hal ini sangat bagus untuk dimiliki: timestamp kejadian, sumber action/event, kejadian dari event/action. Event/action yang dicatat tentu sangat bergantung pada seberapa penting hal itu untuk dicatat tentunya

Audit log bisa jadi merupakan bagian dari normal application log, tetapi yang ditekankan disini adalah mencatat siapa melakukan apa, bukan hanya sekedar apa yang terjadi. Jika memungkinkan, audit log harus dibuat tamper-proof, misal. hanya dapat diakses oleh proses atau user yang memiliki akses khusus dan bukan langsung dari aplikasi. 

## Suspicious Action Throttling and/or blocking

Ini dapat dilihat sebagai generalisasi dari Login Throttling, this time introducing similar mechanics for arbitrary actions that are deemed "suspicious" within the context of the application. For example, an ERP system which allows normal users access to a substantial amount of information, but expects users to be concerned only with a small subset of that information, may limit attempts to access larger than expected datasets too quickly. E.g. prevent users from downloading list of all customers, if users are supposed to work on one or two customers at a time. Note that this is different from limiting access completely—users are still allowed to retrieve information about any customer, just not all of them at once. Depending on the system, throttling might not be enough—e.g. when one invokes an action on all resources with a single request. Then blocking might be required. Note the difference between making 1000 requests in 10 seconds to retrieve full customer information, one customer at a time, and making a single request to retrieve that information at once.

What is suspicious here depends strongly on the expected use of the application. E.g. in one system, deleting 10000 records might be completely legitimate action, but not so in an another one.

## Anonymized Data

Whenever large datasets are exported to third parties, data should be anonymized as much as possible, given the intended use of the data. For example, if a third party service will provide general statistical analysis on a customer database, it probably does not need to know the names, addresses or other personal information for individual customers. Even a generic customer ID number might be too revealing, depending on the data set. Take a look at this article: http://arstechnica.com/tech-policy/2009/09/your-secrets-live-online-in-databases-of-ruin/.

Avoid logging personally identifiable information, for example user’s name.

If your logs contain sensitive information,  make sure you know how logs are protected and where they are located also in the case of cloud hosted log management systems.

If you must log sensitive information try hashing before logging so you can identify the same entity between different parts of the processing.

## Temporary file storage

Make sure you are aware where your application is storing temporary files. If you are using publicly accessible directories (which are most probably the default) like `/tmp` and `/var/tmp`, make sure you create your files with mode 600, so that they are readable only by the user your application is running as. Alternatively, have a protected directory for storing temporary files (directory accessible only by the application user).

## Dedicated vs Shared server environment

The security threats can be quite different depending on whether the application is going to run in a shared or a dedicated environment. Shared here means that there are other (not necessarily 3rd party) applications running on the same server. In that case, having appropriate file permissions becomes critical, otherwise application source code, data files, temporary files, logs, etc might end up accessible by unintended users. Then a security breach in a 3rd party application might result in your application being compromised.

You can never be sure what kind of an environment your application will run for its entire life time—it may start on a dedicated server, but as time goes, 3rd party applications might be added to the same system. That is why it is best to plan from the very first moment that your application runs in a shared environment, and take all precautions. Here's a non-exhaustive list of the files/directories you need to think about:

* application source code
* data directories
* temporary storage directories (often by default the system wide /tmp might be used - see above)
* configuration files
* version control directories - .git, .hg, .svn, etc.
* startup scripts (may contain initialization variables, secrets, etc)
* log files
* crash dumps
* private keys (SSL, SSH, etc)
* etc.

Sometimes, some files need to be accessible by different users (e.g. static content served by apache). In that case, take care to allow only access to what is really needed.

Keep in mind that on a UNIX/Linux filesystem, write access to a directory is permission-wise very powerful—it allows you to delete files in that directory and recreate them (which results in a modified file). /tmp and /var/tmp are by default safe from this effect, because of the sticky bit that should be set on those.

Additionally, as mentioned in the secrets section, file permissions might not be preserved in version control, so even if you set them once, the next checkout/update/whatever may override them. A good idea is then to have a Makefile, a script, a version control hook or something similar that would set the correct permissions when updating the sources.

# Application monitoring

Monitoring the full status of a service requires that both OS-level and application-specific monitoring checks are performed. OS-level checks include, for example, CPU, disk or memory usage, running processes, open ports, etc. Application specific checks are, however, the most important from the point of view of the running service. These can be anything from "does this URL respond and return the HTTP status 200", to checking database connectivity, data consistency, and so on.

This section describes a way to implement the application-specific checks, which would make it easier to monitor the overall application health and give full control to the application developers to determine what checks are meaningful in the context of the concrete application.

In essence, the idea is to have a single endpoint (an application URL) that can give a good status overview of the entire application. This is implemented inside the application and requires work from the project team, but on the other hand, the project team is the one who can really define what is an OK state of the application and what is considered an ERROR state.

The application could implement any number of "subsystem" checks. For example,

* connection to the database is up
* data is in an consistent state (e.g. a list of items in a certain database table is meaningful)
* 3rd party services that the application integrates to are reachable
* ElasticSearch indexes are in a consistent state
* anything else that makes sense for the application

A combined overview status should be provided by the application, aggregating the information from the various subsystem checks. The idea is that an external monitoring system can track only this combined overview, so that the external monitoring does not need to be reconfigured when a new application check is added or modified. Moreover, the developers are the ones that can decide about what the overall status is based on regarding subsystem checks (i.e. which ones are critical, while ones are not, etc).

## Status page

All status checks SHOULD be accessible under `/status` URLs as follows:

* `/status` - the overall status page (mandatory)
* `/status/subsystem1` - a status check for speciffic subsystem (optional)
* ...

The main `/status` page should at a minimum give an overall status of the system, as described in the next section. This means that the main `/status` page should execute ALL subsystem checks and report the aggregated overall system status. It is up to the developers to decide how the overall system status is determined based on the subsystems. For example an `ERROR` state of some non-critical subsystem may only generate an overall `WARNING` status.

For performance reasons, some subsystem checks may be excluded from this overall `/status` page - for example, when the check causes higher resource usage, takes longer time to complete, etc. Overall, the main status page should be light enough so that it can be polled relatively often (every 1-3 minutes) and not cause too much load on the system. Subsystem checks that are excluded from the overall status check should have their own URLs, as shown above. Naturally, monitoring those would require modifications in the monitoring system configuration. To overcome this, a different approach can be taken: the application could perform the heavy subsystem checks in a background process at a rate that is acceptable and store the status internally. This would allow the main status page to reflect also these heavy checks (e.g. it would retrieve the last performed check status). This approach should be used, unless its implementation is too difficult.

## Status page format

We propose two alternative formats for the status pages - `plain` and `JSON`.

### Plain format

The plain format has one status per line in the form `key: value`. The key is a subsystem/check name and the value is the status value. The status value can be one of:

* `OK`
* `WARN Message`
* `ERROR Message`

where `Message` can be some meaningful text, that can help quickly identify the problem. The message is single line, without specified length restriction, but use common sense - e.g. probably should not be longer than 200 characters.

The main status page MUST have a key called `status` that shows the overall aggregated application status. The individual subsystem check status lines are optional. Subsystem status keys should have a suffix `_status`. Here are some examples:

When everything is ok:

```
status: OK
database_status: OK
elastic_search_status: OK
```

When some check is failing:

```
status: ERROR Database is not accessible
database_status: ERROR Connection failed
elastic_search_status: OK
```

Multiple failures at the same time:

```
status: ERROR failed subsystems: database, elasticsearch. For details see https://myapp.example.com/status
database_status: ERROR Connection failed
elastic_search_status: WARN Too few entries in index A.
```

In addition to the status lines, a status page can have non-status keys. For example, those can be showing some metrics (that may or may not be monitored). The additional keys must be prefixed with the subsystem name.

```
status: OK
database_status: OK
database_customers: 378
database_items: 8934748
elastic_search_status: OK
elastic_search_shards: 20
```

The overall status may naturally be based on some of the metrics:

```
status: WARN Too few items in database
database_status: WARN Too few customers in database
database_customers: 378
database_items: 1
elastic_search_status: OK
elastic_search_shards: 20
```

Subsystem checks that have their own URL (`/status/subsystemX`) should follow a similar format, having a mandatory key `status` and a number of optional additional keys. Example for e.g. `/status/database`:

```
status: OK
connection_pool: 30
latency: 2
```

### JSON format

The JSON format of the status pages can be often preferable, for example when the tooling or integration to other systems is easier to achieve via a common data format.

The status values follow the same format as described above - `OK`, `WARN Message` and `ERROR Message`.

The equivalent to the status key form the plain format is a `status` key in the root JSON object. Subsystems should use nested objects also having a mandatory `status` key. Here are some examples:

All is fine:

```json
{
    "status": "OK",
    "database": {
        "status": "OK"
    },
    "elastic_search": {
        "status": "OK"
    }
}
```

Status page with additional metrics:

```json
{
    "status": "OK",
    "uptime": 18234,
    "database": {
        "status": "OK",
        "connection_pool": 30
    },
    "elastic_search": {
        "status": "OK",
        "multinode": false
    }
}
```

Something failing:

```json
{
    "status": "ERROR Database is not accessible. See https://myapp.example.com/status for details.",
    "database": {
        "status": "ERROR Connection failed",
        "connection_timeout": 30
    },
    "elastic_search": {
        "status": "OK"
    }
}
```

## HTTP status codes

Whenever the overall application status is OK, the HTTP status code in the status page response MUST be set to 200 (OK). Otherwise a 5XX error code SHOULD be set. For example, code 500 (Internal Server Error) could be used. Optionally, non-critical WARN status may still respond with 200.

## Load balancer health checks

Often the application is running behind a load balaner. Load balancers typically can monitor application servers by polling a given URL. The health check is used so that the load balancer can stop routing traffic to the failing application servers.

The overall `/status` page is a good candidate for the load balancer health check URL. However, a separate dedicated status page for a load balancer health check provides an important benefit. Such a page can be fine-tuned for when the application is considered to be healthy from the load balancer's perspective. For example, an error in a subsystem may still be considered a critical error for the overall application status, but does not necessarily need to cause the application server to be removed from the load balancer pool. A good example is a 3rd party integration status check. The load balancer health check page should only return non-200 status code when the application instance must be considered non-operational.

The load balancer health check page should be placed at a `/status/health` URL. Depending on your load balancer, the format of that page may deviate from the overall status format described here. Some load balancers may even observe only the returned HTTP status code.

## Access control

The status pages may need proper authorization in place, especially in case they expose debugging information in status messages or application metrics. HTTP basic authentication or IP-based restrictions are usually good enough candidates to consider.

# Checklists

Untuk menghindari melupakan hal-hal yang penting, berikut ini adalah beberapa checklist yang bisa digunakan sebagai checklist.

## Responsibility checklist

Pada projek yang lebih besar, terutama ketika multiple parties terlibat, sangat penting untuk selalu menjaga track dari seluruh role yang berbeda dan tanggungjawabnya. Tabel di bawah ini mengilustrasikan bagaimana list untuk sampai go-live untuk merilis sebuah website: 

| Aspect    | Task                              | Responsible person / party | Deadline     | Status            |
|---        |---                                |---                         |---           |---                |
| Frontend  | Website wireframes                | e.g. Company B / Person X  | e.g. 17.6.   |  e.g. in progress |
| Frontend  | Website design                    | e.g. Company A / Person Z  | e.g. 23.7.   |  e.g. waiting     |
| Frontend  | Website templates                 |   |   |   |
| Frontend  | Content creation and population   |   |   |   |
| Backend   | Setup CMS                         |   |   |   |
| Backend   | Setup staging environment         |   |   |   |
| Backend   | Setup production environment      |   |   |   |
| Backend   | Migrate hosting services to client accounts |   |   |   |
| Backend   | DNS configuration                 |   |   |   |
| Backend   | Setup website analytics           |   |   |   |
| Backend   | Integrate marketing automation    |   |   |   |
| Backend   | Web font license                  |   |   |   |
| Dates     | Website/Product go-live time      |   |   |   |
| Dates     | Publish the website               |   |   |   |

## Release checklist

Ketika sudah siap untuk rilis, ingatlah untuk mengecek semua pada rilis checklist! Hasil yang memuaskan, repeatability and dependability adalah bonus.

You *do* have one, right? If you don't, here is a good generic starting point for you:

* [ ] Deploying works the same no matter which environment you are deploying to
* [ ] All environments have well defined names, and they are referred to using those names
* [ ] All environments have the same underlying software stack
* [ ] All environment configuration is version controlled (web server config, CI build scripts etc.)
* [ ] The product has been tested from the networks from where it will be used (e.g. public Internet, customer LAN)
* [ ] The product has been tested with all of the targeted devices
* [ ] There is a simple way to find out what code is running in any given environment
* [ ] A versioning scheme has been defined
* [ ] Any version of the product should be easily mappable to a state of the code base
* [ ] Rolling back a deployment is possible
* [ ] Backups are running
* [ ] Restoring from a backup has been tested
* [ ] No secrets are stored in version control
* [ ] Logging is turned on
* [ ] There is a well defined process for accessing and searching through logs
* [ ] Logging includes exceptions and stack traces where appropriate
* [ ] Errors can be mapped to stack traces
* [ ] Release notes have been written
* [ ] Server environments are up-to-date
* [ ] A plan for updating the server environments exists
* [ ] The product has been load tested
* [ ] A method exists for replicating the state of one environment in another (e.g. copy prod to QA to reproduce an error)
* [ ] All repeating release processes have been automated

# General questions to consider

* Bagaimana/Apa life-span yang diekspektasikan/dibutuhkan dari projek? 
* Apakah projek merupakan one-off (sekali jadi dan selesai), atau akan ada continous development?
* Apa siklus rilis untuk versi dari servis?
* Apa environments (dev, test, staging, prod, ...) yang ingin di-setup?
* Bagaimana downtime dari production service akan mempengaruhi nilai dari service?
* Seberapa mature teknologi yang digunakan? Apakah major changes merusak backward compatibility diekspektasi?

# Generally proven useful tools

* [HTTPie](https://github.com/jakubroztocil/httpie) adalah tool yang bagus digunakan untuk testing API pada command line. Sangat mudah untuk memasukkan custom headers dan cookies, bahkan di-support oleh session.
* [jq](http://stedolan.github.io/jq/) adalah CLI untuk JSON processor. Massage JSON data masuk dari cURL (atau HTTPie!) kapanpun.

# License

[Futurice Oy](http://www.futurice.com)
Creative Commons Attribution 4.0 International (CC BY 4.0)
