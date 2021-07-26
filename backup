import os
import tarfile
import time
from subprocess import call

import paramiko

backup_config = {
    "user": "root",
    "password": "yourPASS",
    "host": "127.0.0.1",
    "port": "3306",
    "database": "mysql-APP-DB",
    "now": time.strftime('%Y%m%d%H%M%S', time.localtime(time.time())),
    "app_dir": "/opt/someapp",
    "backup_dir": "/opt/someapp/data/",
    "save_copies": 10,
    "save_days": 10,
    "storage": {
        "host": {
            "user": "root",
            "password": "",
            "public_key": "/root/.ssh/id_rsa",
            "ip": "1.1.1.1-YOUR-HOST",
            "ssh_port": "22",
            "target": "/data"
        },
        "s3": {},
        "oss": {},
    }
}


def prepare():
    backup_dir = backup_config.get("backup_dir")
    if not os.path.exists(backup_dir):
        os.makedirs(backup_dir)


def compress_files(name, path):
    with tarfile.open(name, "w:gz") as tar:
        tar.add(path, arcname=os.path.basename(path))


def put_file_over_ssh(src, dst):
    host_config = backup_config.get("storage").get("host")
    hostname = host_config.get("ip")
    port = int(host_config.get("ssh_port")) or paramiko.config.SSH_PORT
    username = host_config.get("user")
    public_key = host_config.get("public_key")
    timeout = 5

    ssh_client = paramiko.SSHClient()
    ssh_client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    ssh_client.connect(hostname, port=port, username=username, key_filename=public_key, timeout=timeout)

    sftp_client = ssh_client.open_sftp()
    sftp_client.put(src, dst)

    sftp_client.close()
    ssh_client.close()


def backup_mysql():
    cli = """mysqldump -u {user} -p"{password}" -h {host} -P {port} \
        -f -R -E --triggers --single-transaction -B \
        {database} > {backup_dir}/backup_{database}_{now}.sql 2>/dev/null""".format(**backup_config)
    call(cli, shell=True)

    sql_name = "{backup_dir}/backup_{database}_{now}.sql".format(**backup_config)
    tgz_name = sql_name + ".tar.gz"
    compress_files(tgz_name, sql_name)


def backup_files():
    app_dir = os.path.abspath(backup_config.get("app_dir"))
    if os.path.islink(app_dir):
        app_dir = os.readlink(app_dir)

    tgz_name = "{backup_dir}/backup_app_{database}_{now}.tar.gz".format(**backup_config)

    compress_files(tgz_name, app_dir)


def scp_files():
    target = backup_config.get("storage").get("host").get("target")
    part1_db = "{backup_dir}/backup_{database}_{now}.sql.tar.gz".format(**backup_config)
    part2_apps = "{backup_dir}/backup_app_{database}_{now}.tar.gz".format(**backup_config)

    for item in [part1_db, part2_apps]:
        if os.path.exists(item):
            put_file_over_ssh(item, os.path.join(target, os.path.basename(item)))


def cleanup():
    backup_dir = backup_config.get("backup_dir")
    save_copies = backup_config.get("save_copies")
    save_days = backup_config.get("save_days")

    if len(os.listdir(backup_dir)) > save_copies * 3:
        for top, dirs, nondirs in os.walk(backup_dir, followlinks=True):
            for item in nondirs:
                fpath = os.path.abspath(os.path.join(top, item))
                st_ctime = os.stat(fpath).st_ctime
                if time.time() - st_ctime > save_days * 86400:
                    os.remove(fpath)


if __name__ == '__main__':
    prepare()

    backup_mysql()
    backup_files()
    scp_files()

    cleanup()