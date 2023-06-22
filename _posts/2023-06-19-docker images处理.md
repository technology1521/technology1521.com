---
layout:  single
title:  "docker images处理"
date:   2023-05-24 
categories:   [kubernetes]
classes: wide
---



在使用Docker构建和管理容器化应用程序时，镜像是一个关键概念。镜像是一个轻量级、可移植的打包格式，它包含了应用程序的代码、运行时环境、系统工具以及所有依赖项。在本文中，我们将讨论如何处理Docker镜像，包括统一导入、上传到Harbor仓库、统一下载与打包，并介绍如何比较仓库镜像与本地镜像。

## 1. 导入镜像

在使用Docker之前，我们通常需要导入所需的镜像。镜像可以从Docker Hub、私有镜像仓库或其他来源获取。要导入一个镜像，可以使用`docker load`命令。

```

$ docker load -i image.tar
```

其中，`image.tar`是要导入的镜像文件。此命令将读取镜像文件并将其加载到本地Docker守护程序中。



将某个目录的所有tar镜像统一导入脚本：

`vim load_dockerimage.sh`

```
#!/bin/bash

# 获取当前目录
current_dir=$1

# 遍历当前目录下的镜像文件
for image_file in ${current_dir}/*.tar; do
  # 导入镜像
  docker load -i "${image_file}"
done
```

执行：

```
chmod +x load_dockerimage.sh
./load_dockerimage.sh "你打包镜像所解压的目录"
```

## 2.标记镜像

在上传docker 镜像之前，我们通常需要标记所需的镜像。要标记一个镜像，可以使用`docker tag`命令。

```
$ docker tag -t "源标签"  "目标标签"
```



以下是一个脚本示例，可以用于为本地所有镜像添加Harbor仓库地址的标签，以便上传到指定的Harbor仓库：

```
#!/bin/bash

# 定义Harbor仓库地址
harbor_address="harbor.example.com"

# 登录到Harbor仓库
docker login ${harbor_address}

# 获取本地所有镜像ID
image_ids=$(docker images -q)

# 遍历镜像ID，为每个镜像添加Harbor仓库地址的标签
for id in ${image_ids}; do
  # 获取镜像的仓库名和标签
  repo_tag=$(docker inspect --format='{{index .RepoTags 0}}' ${id})
  
  # 提取仓库名和标签
  repo=${repo_tag%:*}
  tag=${repo_tag#*:}
  
  # 添加Harbor仓库地址的标签
  harbor_repo="${harbor_address}/${repo}:${tag}"
  
  # 添加标签
  docker tag ${id} ${harbor_repo}
done

# 登出Harbor仓库
docker logout ${harbor_address}
```

在脚本中，你可以通过修改`harbor_address`变量来定义你的Harbor仓库地址。脚本首先登录到指定的Harbor仓库，然后获取本地所有镜像的ID。接下来，它遍历每个镜像ID，提取仓库名和标签，并使用`docker tag`命令为每个镜像添加带有Harbor仓库地址的标签。最后，脚本登出Harbor仓库。

保存脚本为一个文件（例如`tag_images.sh`），并使用以下命令将其设置为可执行文件：

```
$ chmod +x tag_images.sh
```

然后运行脚本即可为本地所有镜像添加Harbor仓库地址的标签：

```
$ ./tag_images.sh
```

请确保在运行脚本之前已经正确安装并配置了Docker，并且能够访问到指定的Harbor仓库。

## 3. 上传到Harbor仓库

Harbor是一个开源的企业级Docker镜像仓库，它提供了镜像存储、复制和分发的功能。如果你有一个私有的Harbor仓库，可以将镜像上传到该仓库以供团队或组织内的其他人使用。

首先，确保你已经登录到Harbor仓库：

```
$ docker login harbor.example.com
```

然后，将本地的镜像标记为Harbor仓库的地址：

```
$ docker tag local-image:tag harbor.example.com/myproject/my-image:tag
```

最后，将标记后的镜像推送到Harbor仓库：

```
$ docker push harbor.example.com/myproject/my-image:tag
```

现在，镜像已经上传到Harbor仓库，并可以在其他地方使用。



以下是一个用于将已经打好标签的镜像上传到Harbor镜像仓库的脚本示例：

```
#!/bin/bash

# 定义Harbor仓库地址和项目名称
harbor_address="harbor.example.com"
project_name="myproject"

# 登录到Harbor仓库
docker login ${harbor_address}

# 获取本地所有已标记的镜像
tagged_images=$(docker images --filter "dangling=false" --format "{{.Repository}}:{{.Tag}}")

# 遍历每个已标记的镜像，将其推送到Harbor仓库
for image in ${tagged_images}; do
  # 构建完整的Harbor镜像地址
  harbor_image="${harbor_address}/${project_name}/${image}"

  # 推送镜像到Harbor仓库
  docker push ${harbor_image}
done

# 登出Harbor仓库
docker logout ${harbor_address}
```

在脚本中，你需要修改`harbor_address`变量为你的Harbor仓库地址，并根据需要设置`project_name`变量为你的项目名称。脚本首先登录到指定的Harbor仓库，然后使用`docker images`命令获取所有已经打好标签的镜像。接下来，它遍历每个已标记的镜像，构建完整的Harbor镜像地址，并使用`docker push`命令将镜像推送到Harbor仓库。最后，脚本登出Harbor仓库。

保存脚本为一个文件（例如`push_to_harbor.sh`），并使用以下命令将其设置为可执行文件：

```

$ chmod +x push_to_harbor.sh
```

然后运行脚本即可将已经打好标签的镜像上传到Harbor镜像仓库：

```
$ ./push_to_harbor.sh
```

请确保在运行脚本之前已经正确安装并配置了Docker，并且能够访问到指定的Harbor仓库。







## 4. 统一下载与打包镜像

有时，我们需要将一个或多个镜像下载到本地或打包成单个文件进行分发。Docker提供了`docker pull`和`docker save`命令来完成这些任务。

要从仓库中下载一个镜像，可以使用`docker pull`命令：

```

$ docker pull harbor.example.com/myproject/my-image:tag
```

该命令将下载指定标签的镜像到本地Docker守护程序。

要将一个或多个镜像打包成单个文件，可以使用`docker save`命令：

```

$ docker save -o image.tar harbor.example.com/myproject/my-image:tag
```

这将把镜像保存为`image.tar`文件。



以下是一个用于将已经打好标签的镜像上传到Harbor镜像仓库的脚本示例：

```
#!/bin/bash

# 定义Harbor仓库地址和项目名称
harbor_address="harbor.example.com"
project_name="myproject"

# 登录到Harbor仓库
docker login ${harbor_address}

# 获取本地所有已标记的镜像
tagged_images=$(docker images --filter "dangling=false" --format "{{.Repository}}:{{.Tag}}")

# 遍历每个已标记的镜像，将其推送到Harbor仓库
for image in ${tagged_images}; do
  # 构建完整的Harbor镜像地址
  harbor_image="${harbor_address}/${project_name}/${image}"

  # 推送镜像到Harbor仓库
  docker push ${harbor_image}
done

# 登出Harbor仓库
docker logout ${harbor_address}
```

在脚本中，你需要修改`harbor_address`变量为你的Harbor仓库地址，并根据需要设置`project_name`变量为你的项目名称。脚本首先登录到指定的Harbor仓库，然后使用`docker images`命令获取所有已经打好标签的镜像。接下来，它遍历每个已标记的镜像，构建完整的Harbor镜像地址，并使用`docker push`命令将镜像推送到Harbor仓库。最后，脚本登出Harbor仓库。

保存脚本为一个文件（例如`push_to_harbor.sh`），并使用以下命令将其设置为可执行文件：

```

$ chmod +x push_to_harbor.sh
```

然后运行脚本即可将已经打好标签的镜像上传到Harbor镜像仓库：

```

$ ./push_to_harbor.sh
```

请确保在运行脚本之前已经正确安装并配置了Docker，并且能够访问到指定的Harbor仓库。



## 5. 对比仓库镜像与本地镜像

有时，我们可能想要比较本地镜像和仓库中的镜像，以确保它们的版本一致。可以使用`docker image ls`命令列出本地已安装的镜像，并使用`docker search`命令在仓库中搜索镜像。

```
$ docker image ls
$ docker search my-image
```

通过比较本地镜像的版本与仓库中镜像的版本，你可以确定是否需要更新本地的镜像或重新下载。



以下是一个用于对比本地镜像和Harbor仓库镜像差异的脚本示例，并将结果输出为txt文件：

```
#!/bin/bash

# 定义Harbor仓库地址和项目名称
harbor_address="harbor.example.com"
project_name="myproject"

# 登录到Harbor仓库
docker login ${harbor_address}

# 获取本地所有镜像
local_images=$(docker images --format "{{.Repository}}:{{.Tag}}")

# 创建输出文件
output_file="image_diff.txt"
> ${output_file}

# 遍历本地镜像，检查与Harbor仓库中对应镜像的差异
for local_image in ${local_images}; do
  # 获取镜像的仓库名和标签
  repo=${local_image%:*}
  tag=${local_image#*:}

  # 构建完整的Harbor镜像地址
  harbor_image="${harbor_address}/${project_name}/${repo}:${tag}"

  # 检查是否存在对应的Harbor镜像
  if docker pull ${harbor_image} &> /dev/null; then
    # 比较镜像的Digest
    local_digest=$(docker inspect --format='{{index .RepoDigests 0}}' ${local_image})
    harbor_digest=$(docker inspect --format='{{index .RepoDigests 0}}' ${harbor_image})

    if [ "${local_digest}" != "${harbor_digest}" ]; then
      # 输出差异到文件
      echo "镜像 ${local_image} 与 Harbor 仓库镜像 ${harbor_image} 不同" >> ${output_file}
      echo "本地 Digest: ${local_digest}" >> ${output_file}
      echo "Harbor Digest: ${harbor_digest}" >> ${output_file}
      echo "" >> ${output_file}
    fi
  else
    # 输出找不到对应Harbor镜像的信息到文件
    echo "找不到对应的 Harbor 仓库镜像：${harbor_image}" >> ${output_file}
    echo "" >> ${output_file}
  fi
done

# 登出Harbor仓库
docker logout ${harbor_address}

# 输出完成信息
echo "镜像差异检查完成，请查看 ${output_file}"
```

在脚本中，你需要修改`harbor_address`变量为你的Harbor仓库地址，并根据需要设置`project_name`变量为你的项目名称。脚本首先登录到指定的Harbor仓库，然后使用`docker images`命令获取本地所有镜像的列表。接下来，它遍历每个本地镜像，检查与Harbor仓库中对应镜像的差异。脚本通过比较镜像的Digest来判断差异。如果镜像在Harbor仓库中不存在，脚本将输出相应信息。最后，脚本将差异信息输出到指定的txt文件中，并在完成时显示输出文件的路径。





保存脚本为一个文件（例如`compare_images.sh`），并使用以下命令将其设置为可执行文件：

```

$ chmod +x compare_images.sh
```

然后运行脚本即可对比本地镜像和Harbor仓库镜像的差异，并将结果输出为txt文件：

```

$ ./compare_images.sh
```

请确保在运行脚本之前已经正确安装并配置了Docker，并且能够访问到指定的Harbor仓库。最后，你可以打开生成的txt文件以查看镜像差异信息。

## 6.定时清除本地`none`镜像



以下是一个定时清除本地`none`镜像的脚本示例，你可以使用cron任务来定期执行该脚本：

```
#!/bin/bash

# 清除本地none镜像
docker rmi $(docker images -q --filter "dangling=true") &> /dev/null
```

将上述脚本保存为一个文件（例如`cleanup_images.sh`），然后使用以下命令将其设置为可执行文件：

```

$ chmod +x cleanup_images.sh
```

接下来，使用cron任务来定期执行该脚本。打开cron表编辑器：

```

$ crontab -e
```

在编辑器中添加以下行，以在每天凌晨3点清除本地`none`镜像：

```
0 3 * * * /path/to/cleanup_images.sh
```

保存并退出编辑器。现在，脚本将在每天凌晨3点自动清除本地的`none`镜像。

请注意，脚本中使用的`docker rmi`命令将删除所有`none`镜像，包括未被使用的中间层镜像和临时构建镜像。确保在执行脚本之前，你不再需要这些`none`镜像。

## 结论

处理Docker镜像是容器化应用程序开发和管理过程中的关键任务。通过统一导入镜像、上传到Harbor仓库、统一下载与打包镜像，并比较仓库镜像与本地镜像，我们可以更好地管理和协作开发容器化应用程序。Docker的强大功能为我们提供了灵活性和便捷性，使得构建和部署容器化应用程序变得更加高效。

希望本文对你理解和处理Docker镜像有所帮助！如果你有任何问题或疑问，请随时提问。


