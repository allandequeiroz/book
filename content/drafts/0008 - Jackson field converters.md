+++
title = "Jackson field converters"
description = ""
tags = [
    "java",
    "jackson"
]
date = "2014-04-02"
categories = [
    "software",
    "devops"
]
draft = true
+++

# Custom serialisation with Jackson at field level
Quick one to not forget since is quite useful. How to specify custom serializers at field level.

The example here is a bit ridiculous but a few days ago I needed it for a real scenario and was quite hand so, ignore the example since I couldn't think about anything better and keep your focus on how handy it may be.

### Model
<script src="https://gist.github.com/allandequeiroz-snippets/369da91a57f9fefd8a570b50ef96c978.js"></script>

### Serializer
<script src="https://gist.github.com/allandequeiroz-snippets/62dc5a986518576b34525337f0e4dcd3.js"></script>

### Deserializer
<script src="https://gist.github.com/allandequeiroz-snippets/f6aba3554e0cf0d25f8934c2747ec1e1.js"></script>

### Test
<script src="https://gist.github.com/allandequeiroz-snippets/b8ebb5f184ddbef2810fa266044f135c.js"></script>

### Result
![Cloud Kubernetes Block](http://url/to/img.png) 