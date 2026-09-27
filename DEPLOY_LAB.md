# Deploying in the IBM Skills Network lab

Run these in order. Steps 1–2 run in the **Code Engine** lab environment. Steps 3–9 run in
the **Kubernetes** lab environment, and the Node app from step 4 must keep running while
you use the deployed site.

## 1. Clone the repo

```bash
git clone https://github.com/ShahbazShaddy/xrwvm-fullstack_developer_capstone.git
```

## 2. Deploy the sentiment analyzer to Code Engine

In the Code Engine lab, click **Create Project** first, then:

```bash
cd xrwvm-fullstack_developer_capstone/server/djangoapp/microservices
docker build . -t us.icr.io/${SN_ICR_NAMESPACE}/senti_analyzer
docker push us.icr.io/${SN_ICR_NAMESPACE}/senti_analyzer
ibmcloud ce application create --name sentianalyzer \
  --image us.icr.io/${SN_ICR_NAMESPACE}/senti_analyzer \
  --registry-secret icr-secret --port 5000
```

The container listens on port **5000** (`flask run`'s default), so `--port 5000` is required.
Copy the URL that `create` prints (or run `ibmcloud ce application get --name sentianalyzer --output url`)
and check it:

```bash
curl "<sentianalyzer-url>/analyze/Fantastic%20services"   # {"sentiment": "positive"}
```

## 3. Clone the repo in the Kubernetes lab

```bash
cd /home/project
git clone https://github.com/ShahbazShaddy/xrwvm-fullstack_developer_capstone.git
```

## 4. Start MongoDB + the Node API with docker compose

```bash
cd /home/project/xrwvm-fullstack_developer_capstone/server/database
docker build . -t nodeapp
docker-compose up -d
curl localhost:3030/fetchDealers | head -c 200
```

Open **Skills Network Toolbox → Other → Launch Application**, enter port **3030**, and copy the
URL. It looks like `https://<user>-3030.<...>.proxy.cognitiveclass.ai`.

## 5. Create the Django `.env` with the two URLs

```bash
cd /home/project/xrwvm-fullstack_developer_capstone/server/djangoapp
cat > .env <<EOF
backend_url=https://<user>-3030.<...>.proxy.cognitiveclass.ai
sentiment_analyzer_url=https://sentianalyzer.<...>.codeengine.appdomain.cloud/
EOF
cat .env
```

A trailing slash is optional on both URLs, because `restapis.py` normalises them.
`.env` is gitignored, but it is included in the Docker build context, so the image picks it up.

## 6. Build and push the dealership image

The Dockerfile builds the React frontend itself (multi-stage), so no `npm` step is needed.

```bash
MY_NAMESPACE=$(ibmcloud cr namespaces | grep sn-labs-)
echo $MY_NAMESPACE

cd /home/project/xrwvm-fullstack_developer_capstone/server
docker build -t us.icr.io/$MY_NAMESPACE/dealership .
docker push us.icr.io/$MY_NAMESPACE/dealership
```

## 7. Fill the namespace in `deployment.yaml`

```bash
sed -i "s/<namespace>/$MY_NAMESPACE/" deployment.yaml
grep image: deployment.yaml   # us.icr.io/sn-labs-<you>/dealership
```

## 8. Apply the deployment

```bash
kubectl apply -f deployment.yaml
kubectl get pods -l run=dealership   # wait for STATUS Running
```

## 9. Port-forward 8000

```bash
kubectl port-forward deployment.apps/dealership 8000:8000
```

Then open **Launch Application** on port **8000**. The site is served at
`https://<user>-8000.<...>.proxy.cognitiveclass.ai`. Django already allows that host
(`ALLOWED_HOSTS` includes `.proxy.cognitiveclass.ai`, and `CSRF_TRUSTED_ORIGINS` includes
`https://*.cognitiveclass.ai`).

## Notes

- **Admin user.** The pod starts with a fresh SQLite database, because `entrypoint.sh` runs
  migrations on start. If you need the admin, open a second terminal and run
  `kubectl exec -it deployment.apps/dealership -- python manage.py createsuperuser`.
  The account is lost if the pod restarts.
- **Redeploying after a change.** Rebuild and push (step 6), then run
  `kubectl rollout restart deployment dealership`. `imagePullPolicy: Always` pulls the new
  image.
- **Empty dealer list.** If the dealer list is empty, the Node app from step 4 has stopped,
  or `backend_url` doesn't match the 3030 Launch Application URL.
