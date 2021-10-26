---
title: 使用Python编写优雅的命令行程序
date: 2020-07-23 09:09:35
categories:
- 运维
tags: 
- Python Cli Application
---

编写脚本时想要读取参数，最原始的方法就是通过sys模块获取参数列表，当读取一到两个简单参数时并没有什么不妥，但是当参数变得复杂多样时则会变得难以维护和扩展

```python
import sys


def main():
    """
     通过sys模块来识别参数demo, http://blog.csdn.net/ouyang_peng/
    """
    print('参数个数为:', len(sys.argv), '个参数。')
    print('参数列表:', str(sys.argv))
    print('脚本名为：', sys.argv[0])
    for i in range(1, len(sys.argv)):
        print('参数 %s 为：%s' % (i, sys.argv[i]))


if __name__ == "__main__":
    main()
```

#### argparse

好在功能强大的python内置了argparse库可以让我们更方便的读取参数

```python
import argparse
import hashlib
import bcrypt
import hmac


def md5_cypher(args):
    md5_hash = hashlib.md5()
    md5_hash.update(args.password.encode('utf-8'))
    return md5_hash.hexdigest()

def sha256_cypher(args):
    sha256_hash = hashlib.sha256()
    sha256_hash.update(args.password.encode('utf-8'))
    return sha256_hash.hexdigest()

def hmac_md5_cypher(args):
    if not args.key:
        return 'error: the following arguments are required: key'
    hmac_md5_hash = hmac.new(args.key.encode('utf-8'), args.password.encode('utf-8'), digestmod='MD5')
    return hmac_md5_hash.hexdigest()

def bcrypt_cypher(args):
    return bcrypt.hashpw(args.password, bcrypt.gensalt(args.salt_len))

switch = {
    'md5': md5_cypher,
    'sha256': sha256_cypher,
    'hmac-md5': hmac_md5_cypher,
    'bcrypt': bcrypt_cypher,
}

def cypher():
    parser = argparse.ArgumentParser(description='密码加密,支持:md5 sha256 hmac-md5 bcrypt等方式')
    group = parser.add_mutually_exclusive_group()
    group.add_argument('-t', '--hash-type', choices=['md5', 'sha256', 'hmac-md5', 'bcrypt'])
    parser.add_argument('-k', '--key', type=str, dest='hmac算法中的key')
    parser.add_argument('--salt-len', type=int, default=10, dest='迭代次数,默认为10次')
    parser.add_argument('password')
    args = parser.parse_args()
    
    try:
        cyphertext = switch[args.type](args)
        print(cyphertext)
    except KeyError as e:
        pass


if __name__ == '__main__':
    cypher()
```

当然除了内置的参数解析模块也有需求优秀的第三方模块，相比argparse第三方库一般都支持注解方式，参数校验也更为丰富

#### typer

> [https://typer.tiangolo.com/](https://typer.tiangolo.com/)

```python
import typer
import hashlib
import bcrypt
import hmac

app = typer.Typer()

def md5_cypher(args):
    md5_hash = hashlib.md5()
    md5_hash.update(args['password'].encode('utf-8'))
    return md5_hash.hexdigest()

def sha256_cypher(args):
    sha256_hash = hashlib.sha256()
    sha256_hash.update(args['password'].encode('utf-8'))
    return sha256_hash.hexdigest()

def hmac_md5_cypher(args):
    if not args['key']:
        return typer.echo(typer.style('error: the following arguments are required: key', fg=typer.colors.MAGENTA))
        raise typer.Exit()
    hmac_md5_hash = hmac.new(args['key'].encode('utf-8'), args['password'].encode('utf-8'), digestmod='MD5')
    return hmac_md5_hash.hexdigest()

def bcrypt_cypher(args):
    return bcrypt.hashpw(args['password'], bcrypt.gensalt(args['salt_len']))

switch = {
    'md5': md5_cypher,
    'sha256': sha256_cypher,
    'hmac-md5': hmac_md5_cypher,
    'bcrypt': bcrypt_cypher,
}

hash_types = ['md5', 'sha256', 'hmac-md5', 'bcrypt']

def hash_type(value):
    if value not in hash_types:
        raise typer.BadParameter(typer.style('hash-type 应该在' + str(hash_types) + '中选择', fg=typer.colors.MAGENTA))
    return value

def salt_len(value: int):
    if value > 15 or value < 5:
        raise typer.BadParameter(typer.style('salt-len 应该在5~15之间', fg=typer.colors.MAGENTA))
    return value

@app.command()
def cypher(
    password: str = typer.Option(
        ..., prompt=True, hide_input=True
    ),
    hash_type: str = typer.Option(..., callback=hash_type), 
    key: str = typer.Option('abc@123'), 
    salt_len: int = typer.Option(default=10, callback=salt_len)
):
    args = {
        'hash_type': hash_type,
        'key': key,
        'salt_len': salt_len,
        'password': password,
    }
    try:
        cyphertext = switch[hash_type](args)
        typer.echo(typer.style(cyphertext, fg=typer.colors.WHITE, bold=True))
    except KeyError as e:
        pass


if __name__ == "__main__":
    app()
```

#### click

> [https://palletsprojects.com/p/click/](https://palletsprojects.com/p/click/)

```python
import click
import hashlib
import bcrypt
import hmac


def md5_cypher(args):
    md5_hash = hashlib.md5()
    md5_hash.update(args['password'].encode('utf-8'))
    return md5_hash.hexdigest()

def sha256_cypher(args):
    sha256_hash = hashlib.sha256()
    sha256_hash.update(args['password'].encode('utf-8'))
    return sha256_hash.hexdigest()

def hmac_md5_cypher(args):
    if not args['key']:
        return click.secho('error: the following arguments are required: key', bg='black', fg='red')
    hmac_md5_hash = hmac.new(args['key'].encode('utf-8'), args['password'].encode('utf-8'), digestmod='MD5')
    return hmac_md5_hash.hexdigest()

def bcrypt_cypher(args):
    return bcrypt.hashpw(args['password'], bcrypt.gensalt(args['salt_len']))

switch = {
    'md5': md5_cypher,
    'sha256': sha256_cypher,
    'hmac-md5': hmac_md5_cypher,
    'bcrypt': bcrypt_cypher,
}

@click.command()
@click.option(
    '-t', '--hash-type',
    type=click.Choice(['md5', 'sha256', 'hmac-md5', 'bcrypt'], 
    case_sensitive=False)
)
@click.option(
    '-k', '--key',
    type=str,
    required=False
)
@click.option(
    '--salt-len',
    type=click.IntRange(5, 15),
    default=10,
    required=False
)
@click.option(
    '--password', 
    prompt=True, 
    hide_input=True
)
def cypher(hash_type, key, salt_len, password):
    args = {
        'hash_type': hash_type,
        'key': key,
        'salt_len': salt_len,
        'password': password,
    }
    try:
        cyphertext = switch[hash_type](args)
            
        click.echo(cyphertext)
    except KeyError as e:
        pass

if __name__ == '__main__':
    cypher()
```

#### fire

> [https://github.com/google/python-fire](https://github.com/google/python-fire)

```python
import fire
import hashlib
import bcrypt
import hmac

class Cypher(object):
    """密码加密,支持:md5 sha256 hmac-md5 bcrypt等方式"""
    def md5(self, password):
        md5_hash = hashlib.md5()
        md5_hash.update(password.encode('utf-8'))
        return md5_hash.hexdigest()
    
    def sha256(self, password):
        sha256_hash = hashlib.sha256()
        sha256_hash.update(password.encode('utf-8'))
        return sha256_hash.hexdigest()
    
    def hmac_md5(self, password, key):
        hmac_md5_hash = hmac.new(key.encode('utf-8'), password.encode('utf-8'), digestmod='MD5')
        return hmac_md5_hash.hexdigest()
    
    def bcrypt_cypher(self, password, salt_len):
        return bcrypt.hashpw(password, bcrypt.gensalt(salt_len))

if __name__ == "__main__":
    fire.Fire(Cypher)
```

