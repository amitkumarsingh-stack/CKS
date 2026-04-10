**Question 1:**
A deployment named ``fruits`` in the namespace salad has three containers:
* apple
* banana
* kiwi
One of these containers has the package **curl** installed. Identify which container has that package from the running containers, and create an **SBOM SPDX** for the container's image.

Use the tarball archive of that particular image stored under ``/root/ImageTarballs`` directory for generating the SPDX JSON. The archives have names matching their images.

Save the output in ``~/bugged-fruit.spdx``. Save the container name in ``~/bugged-container.txt``.

**Note:** bom and all its required dependencies are already installed.

## Solution

**Step 1: First, check the deployments running in the salad namespace by executing the following command**
```
kubectl get deployments -n salad
```

**Step 2: run the following command in succession to fetch the container name and save it to a text file**
```
kubectl exec -n salad fruits-<string> -c apple -- apk info | grep curl && echo apple > ~/bugged-container.txt

kubectl exec -n salad fruits-<string> -c banana -- apk info | grep curl && echo banana > ~/bugged-container.txt

kubectl exec -n salad fruits-<string> -c kiwi -- apk info | grep curl && echo kiwi > ~/bugged-container.txt
```
Please repeat this process for both the kiwi and banana containers. In our case ``apple`` is the container which has the curl package.

**Step 3: To determine the image name used in the container, describe the pod and check the Containers section in the output using the command:**
```
kubectl describe pod fruits-54665c68db-mcwg9 -n salad
```

**Step 4: After identifying the image name, navigate to the /root/ImageTarballs/ directory to locate the tarball associated with that image.**
```
cat /root/imageTarballs
```

**Step 5: Finally, execute the following command to generate the SPDX JSON:**
```
bom generate --image-archive /root/ImageTarballs/<image_name>.tar --format json --output ~/bugged-fruit.spdx
```
Refer document https://github.com/kubernetes-sigs/bom?tab=readme-ov-file#bom-generate

