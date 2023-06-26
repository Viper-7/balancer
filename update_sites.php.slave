#!/usr/bin/php
<?php
$db = new PDO('mysql:host=dbhost:dbport;dbname=dbname', 'dbuser', 'dbpass');
$db->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);

ConfigBuilder::setupDB($db);

$cb = new ConfigBuilder($db);
$cb->build();

class ConfigBuilder {
    public $sites = array();
    public $sitePath = '/etc/nginx/http.d/';

    protected function buildHTTPConfig($site) {
        echo "Building config for {$site['key']}\n";
        $config = "server {\n";
        $config .= "    listen 80;\n";
        foreach($site['hosts'] as $host) {
            $config .= "    server_name {$host['hostname']};\n";
            if(isset($host['paths'])) {
                foreach($host['paths'] as $path) {
                    $config .= "    location {$path['path']} {\n";
                    $config .= "        proxy_pass {$path['target']['target_proto']}://{$path['target']['target_host']}:{$path['target']['target_port']};\n";
                    $config .= "    }\n";
                }
            }
            $config .= "    location / {\n";
            $config .= "        proxy_pass {$host['target']['target_proto']}://{$host['target']['target_host']}:{$host['target']['target_port']};\n";
            $config .= "    }\n";
        }
        $config .= "}\n";

        return $config;
    }

    protected function buildHTTPSConfig($site) {
        $config = "server {\n";
        foreach($site['hosts'] as $host) {
            $config .= "    server_name {$host['hostname']};\n";
            if(isset($host['paths'])) {
                foreach($host['paths'] as $path) {
                    $config .= "    location {$path['path']} {\n";
                    $config .= "        proxy_pass {$path['target']['target_proto']}://{$path['target']['target_host']}:{$path['target']['target_port']};\n";
                    $config .= "    }\n";
                }
            }
            $config .= "    location / {\n";
            $config .= "        proxy_pass {$host['target']['target_proto']}://{$host['target']['target_host']}:{$host['target']['target_port']};\n";
            $config .= "    }\n";
        }
        $config .= "\n";
        $config .= "    listen 443 ssl; # managed by Certbot\n";
        $config .= "    ssl_certificate /etc/letsencrypt/live/{$site['key']}/fullchain.pem; # managed by Certbot\n";
        $config .= "    ssl_certificate_key /etc/letsencrypt/live/{$site['key']}/privkey.pem; # managed by Certbot\n";
        $config .= "    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot\n";
        $config .= "    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot\n";
        $config .= "\n}\n";
        $config .= "server {\n";
        $config .= "    if (\$host = {$site['key']}) {\n";
        $config .= "        return 301 https://\$host\$request_uri;\n";
        $config .= "    } # managed by Certbot\n";
        $config .= "\n";
        $config .= "\n";
        $config .= "    listen 80;\n";
        $config .= "    server_name {$site['key']};\n";
        $config .= "    return 404; # managed by Certbot\n";
        $config .= "\n\n}\n";

        return $config;
    }

    protected function compareConfig($site) {
        echo "Comparing config for {$site['key']}: ";
        $sitefile = $this->sitePath . $site['key'];
        if(!file_exists($sitefile) || trim(file_get_contents($sitefile)) != trim($this->buildHTTPSConfig($site))) {
            file_put_contents($sitefile, $this->buildHTTPConfig($site));
            exec('service nginx reload');
            $domains = array();
            foreach($site['hosts'] as $host) {
                $domains[] = $host['hostname'];
            }
            //exec('certbot --nginx -n -d ' . implode(' -d ', $domains));

            $config = $this->buildHTTPSConfig($site);
            file_put_contents($sitefile, $config);

            if(trim(file_get_contents($sitefile)) != trim($config)) {
                echo "Alert! Fresh config differs from expected!\n";
                echo "Generated:\n";
                echo $config . "\n";
                echo "Filesystem:\n";
                echo trim(file_get_contents($sitefile)) . "\n";
                return false;
            }

            echo "Config has been updated\n";
            return true;
        }

        echo "Up to date\n";
        return true;
    }

    public static function setupDB($db) {
        try {
            $stmt = $db->prepare('SELECT * FROM sites');
            $stmt->execute();
        } catch(PDOException $e) {
            if($e->getMessage() == 'SQLSTATE[42S02]: Base table or view not found: 1146 Table \'balancer.sites\' doesn\'t exist') {
                $db->exec("CREATE TABLE if not exists sites (
                    id INT NOT NULL AUTO_INCREMENT,
                    `key` VARCHAR(500),
                    default_target_id INT,
                    PRIMARY KEY (id)
                  );

                  CREATE TABLE if not exists targets (
                    id INT NOT NULL AUTO_INCREMENT,
                    target_host VARCHAR(500) NOT NULL,
                    target_port VARCHAR(10) DEFAULT '80',
                    target_proto VARCHAR(50) DEFAULT 'http',
                    name VARCHAR(500),
                    PRIMARY KEY (id)
                  );

                  CREATE TABLE if not exists hosts (
                    id INT NOT NULL AUTO_INCREMENT,
                    site_id INT NOT NULL,
                    target_id INT NOT NULL,
                    hostname VARCHAR(500) NOT NULL,
                    PRIMARY KEY (id),
                    FOREIGN KEY (site_id) REFERENCES sites(id),
                    FOREIGN KEY (target_id) REFERENCES targets(id)
                  );

                  CREATE TABLE if not exists paths (
                    id INT NOT NULL AUTO_INCREMENT,
                    site_id INT NOT NULL,
                    host_id INT NOT NULL,
                    target_id INT NOT NULL,
                    path VARCHAR(500),
                    PRIMARY KEY (id),
                    FOREIGN KEY (site_id) REFERENCES sites(id),
                    FOREIGN KEY (host_id) REFERENCES hosts(id),
                    FOREIGN KEY (target_id) REFERENCES targets(id)
                  );
                  ");
            }
            $stmt = $db->prepare('SELECT * FROM sites');
            $stmt->execute();
        }
    }

    public function __construct($db) {
        $this->db = $db;

        $stmt = $this->db->prepare('SELECT * FROM sites');
        $stmt->execute();

        foreach($stmt->fetchAll() as $site) {
            $targets = $this->db->prepare('SELECT * FROM targets');
            $targets->execute();
            foreach($targets->fetchAll() as $target) {
                $site['targets'][$target['id']] = $target;
            }

            $site['target'] = $site['targets'][$site['default_target_id']];
            $this->sites[$site['key']] = $site;

            $hosts = $this->db->prepare('SELECT * FROM hosts WHERE site_id = ?');
            $hosts->execute(array($site['id']));
            foreach($hosts->fetchAll() as $host) {
                $host['target'] = $this->sites[$site['key']]['targets'][$host['target_id']];
                $this->sites[$site['key']]['hosts'][$host['id']] = $host;

                $paths = $this->db->prepare('SELECT * FROM paths WHERE host_id = ?');
                $paths->execute(array($host['id']));
                foreach($paths->fetchAll() as $path) {
                    $path['target'] = $this->sites[$site['key']]['targets'][$path['target_id']];
                    $this->sites[$site['key']]['hosts'][$host['id']]['paths'][$path['id']] = $path;
                }
            }
        }
    }

    public function build() {
        foreach($this->sites as $site) {
            $this->compareConfig($site);
        }
    }
}
