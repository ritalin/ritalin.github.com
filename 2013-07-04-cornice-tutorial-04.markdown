	---
layout: post
title: "Cornice Tutorial (3)"
date: 2013-06-12 21:50
comments: true
categories: python cornice
---

## はじめに
公式サイトの[チュートリアル](http://cornice.readthedocs.org/en/latest/tutorial.html)の写経、第肆回。
ユーザー管理サービスの実装において、Validatorの話がでてくるので、内容を抑えつつ一覧、追加、削除の実装を進めていきたいと思います。

<!-- more -->

# ユーザー削除の実装
前回手をつけられなかった、ユーザー削除に手をつけておく

まずは存在チェックのValidation

```
def validate_token(request):
    header = 'X-Messaging-Token'
    token = request.headers.get(header)
    if token is None:
        raise _401()

    token = token.split('-')
    if len(token) != 2:
        raise _401()

    user, token = token

    if not (user in _USERS and _USERS[user] == token):
        raise _401()

    request.validated['user'] = user

class _401(exc.HTTPError):
    def __init__(self, message = 'Unauthorized'):
        body = { 'status': 401, 'message': message}
        Response.__init__(self, json.dumps(body))

        elf.status = 401
        self.content_type = 'application/json'        
```

次いで削除部分本体

```
@users.delete(validators=validate_token)
def del_user(request):
    """remove the user
    """
```





