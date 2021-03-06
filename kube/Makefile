SHELL = /bin/bash

# 变量
export NAMESPACE   := rails6
export PJNAME         := rails6
export IMAGE_NAME  := huangxiaofenglogin/rails6-puma

# docker 镜像标签
ifeq ($(tag),)
	tag := 1.0.2
else
	tag := $(tag)
endif

#<<< 通用大方法 >>>

# 开启 k8s 服务：make up tag=1.0.1
# 注意：每执行一次要升级一个版本号
up:
	$(MAKE) delete; echo ""
	$(MAKE) namespace; echo ""
	$(MAKE) docker-build tag=$(tag); echo ""
	$(MAKE) docker-push tag=$(tag); echo ""
	$(MAKE) apply; sleep 3; echo "";
	$(MAKE) all; echo ""
	$(MAKE) pod

# 更新镜像代码并替换 pod 里的旧镜像：make update tag=1.0.1
# 注意：每执行一次要升级一个版本号
update:
	$(MAKE) docker-build tag=$(tag); echo ""
	$(MAKE) docker-push tag=$(tag); echo ""
	$(MAKE) set-image tag=$(tag); echo ""
	$(MAKE) all; echo ""
	$(MAKE) pod

# 清空命名空间内容
down: 
	$(MAKE) delete; echo ""
	$(MAKE) http-delete; echo ""
	$(MAKE) https-delete; echo ""
	$(MAKE) all

# 删除所有（包括命名空间和 ingress controller）
remove:
	$(MAKE) delete; echo ""
	$(MAKE) http-delete; echo ""
	$(MAKE) https-delete; echo ""
	kubectl delete  -f ingresses/ing_controller.yaml; echo ""
	kubectl delete namespaces $(NAMESPACE) --ignore-not-found; echo ""
	$(MAKE) all

#<<< 通用小方法 >>>

namespace:
	if [ "$$(kubectl get namespaces | grep $(NAMESPACE))" == "" ]; then \
		kubectl create namespace $(NAMESPACE); \
	fi

# 本地不存在的镜像版本，才会执行 build
docker-build:
	if [ "$$(docker image ls -q $(IMAGE_NAME):$(tag))" == "" ]; then \
		docker build -t $(IMAGE_NAME):$(tag)   ..; \
	fi; \
	docker history $(IMAGE_NAME):$(tag) 2>&1 | head -n 6

docker-push:
	if [ "$$(docker image ls -q $(IMAGE_NAME):$(tag))" != "" ]; then \
		docker push $(IMAGE_NAME):$(tag); \
	fi

apply:
	cat deployments/*.yaml | kubectl -n $(NAMESPACE) apply -f -
	cat services/*.yaml       | kubectl -n $(NAMESPACE) apply -f -
	cat env/*.yaml             | kubectl -n $(NAMESPACE) apply -f -
	cat jobs/*.yaml            | kubectl -n $(NAMESPACE) apply -f -
	cat volumes/*.yaml       | kubectl -n $(NAMESPACE) apply -f -

delete:
	cat deployments/*.yaml | kubectl -n $(NAMESPACE) delete -f - --ignore-not-found
	cat services/*.yaml       | kubectl -n $(NAMESPACE) delete -f - --ignore-not-found
	cat env/*.yaml             | kubectl -n $(NAMESPACE) delete -f - --ignore-not-found
	cat jobs/*.yaml            | kubectl -n $(NAMESPACE) delete -f - --ignore-not-found
	cat volumes/*.yaml       | kubectl -n $(NAMESPACE) delete -f - --ignore-not-found

set-image:
	kubectl -n $(NAMESPACE) set image deployments/$(PJNAME)-puma puma=$(IMAGE_NAME):$(tag)
	kubectl -n $(NAMESPACE) set image deployments/$(PJNAME)-sidekiq sidekiq=$(IMAGE_NAME):$(tag)

# 有时会连接失败，重试几次
http:
	$(MAKE) http-delete; echo ""
	kubectl apply  -f ingresses/ing_controller.yaml; echo ""
	for i in {1..10}; do \
		kubectl -n $(NAMESPACE) apply  -f ingresses/ing_http.yaml && break || \
		echo "重试$(i)/10" && sleep 2; \
	done

http-delete:
	kubectl -n $(NAMESPACE) delete  -f ingresses/ing_http.yaml --ignore-not-found

https:
	if ls env/_chain.crt env/_key.key >/dev/null 2>&1; then \
		$(MAKE) https-delete; \
		kubectl apply  -f ingresses/ing_controller.yaml; \
		kubectl -n $(NAMESPACE) apply -f ingresses/ing_https.yaml; \
		kubectl -n $(NAMESPACE) create secret tls $(PJNAME)-tls --key env/_key.key --cert env/_chain.crt; \
	fi

https-delete:
	kubectl -n $(NAMESPACE) delete -f ingresses/ing_https.yaml --ignore-not-found
	kubectl -n $(NAMESPACE) delete secret $(PJNAME)-tls --ignore-not-found

# 查看集群状态
all:
	kubectl -n $(NAMESPACE) get all,pv,pvc,ns -o wide

# 实时监控 pod 状态
pod:
	for i in {1..10000}; do \
		kubectl -n $(NAMESPACE) get pod -o wide; sleep 3; echo ""; \
	done

#<<< 操作 pod >>>

# 进入 puma 的最新 pod
exec:
	kubectl -n $(NAMESPACE) exec -it "$$(kubectl -n $(NAMESPACE) get pods -o=custom-columns=NAME:.metadata.name | grep puma | head -1)" sh

# 查看 puma 最新 pod 的日志
log:
	kubectl -n $(NAMESPACE) logs "$$(kubectl -n $(NAMESPACE) get pods -o=custom-columns=NAME:.metadata.name | grep puma | head -1)"

# 复制进去：make cpi f1=config/cable.yml f2=.
cpi:
	kubectl -n $(NAMESPACE) cp ../$(f1) "$$(kubectl -n $(NAMESPACE) get pods -o=custom-columns=NAME:.metadata.name | grep puma | head -1):$(f2)"

# 复制出来：make cpo f1=Gemfile f2=tmp/Gemfile
cpo:
	kubectl -n $(NAMESPACE) cp "$$(kubectl -n $(NAMESPACE) get pods -o=custom-columns=NAME:.metadata.name | grep puma | head -1):$(f1)" ../$(f2)
