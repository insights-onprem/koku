# Insights OnPrem instructions for starting koku in development environment
## Kafka
We use the ros-ocp-backend kafka container. Start the ros-ocp-backend kafka service.
- clone https://github.com/insights-onprem/ros-ocp-backend
- go to folder deployment/docker-compose
- execute `docker-compose up -d kafka kafka-create-topics`

## Build and start
- copy .env.example to new .env file
- execute `docker-compose build`
- execute `docker-compose up -d`. some services fail the first time - restart them

## Create S3 buckets
Go to `http://localhost:9090`. login with credentials from `.env`. Look for `S3_ACCESS_KEY` and `S3_ACCESS_KEY`. Create two buckets: koku-bucket and ocp-ingress (configurable via `S3_BUCKET_NAME_OCP_INGRESS` and `S3_BUCKET_NAME`). Change access policy to public

## Create source
Execute the following command (koku-server handles it):
```
curl -X POST "http://localhost:8000/api/cost-management/v1/sources/"   -H "X-Rh-Identity: ewogICJpZGVudGl0eSI6IHsKICAgICJ0eXBlIjogIlVzZXIiLAogICAgImFjY291bnRfbnVtYmVyIjogIjAwMDAwMDEiLAogICAgInVzZXIiOiB7CiAgICAgICJ1c2VybmFtZSI6ICJ1c2VyX2RldiIsCiAgICAgICJlbWFpbCI6ICJ1c2VyX2RldkBmb28uY29tIiwKICAgICAgImlzX29yZ19hZG1pbiI6IHRydWUsCiAgICAgICJhY2Nlc3MiOiB7fQogICAgfQogIH0sCiAgIm9yZ19pZCI6ICIwMDAwMDEiLAogICJlbnRpdGxlbWVudHMiOiB7CiAgICAiY29zdF9tYW5hZ2VtZW50IjogewogICAgICAiaXNfZW50aXRsZWQiOiB0cnVlCiAgICB9CiAgfQp9Cg=="   -H "Content-Type: application/json"   -d '{                                                                                                           "name": "New OpenShift Cluster",                                                                                                    "source_type": "OCP",
"authentication": {
  "credentials": {
    "cluster_id": "023d9b0e-7ca6-481d-b04f-ea606becd588"
  }
 },
 "billing_source": {
   "data_source": {}
 }
}'
```

## Test
- Upload the file `ros-ocp-backend/scripts/samples/cost-mgmt.tar.gz` to ocp-ingress bucket
- Send a message to Kafka:
```
echo -e 'service:hccm\t{"request_id": "test-no-retention-802", "url": "http://minio:9000/ocp-ingress/cost-mgmt.tar.gz", "b64_identity": "ewogICJpZGVudGl0eSI6IHsKICAgICJ0eXBlIjogIlVzZXIiLAogICAgImFjY291bnRfbnVtYmVyIjogIjAwMDAwMDEiLAogICAgInVzZXIiOiB7CiAgICAgICJ1c2VybmFtZSI6ICJ1c2VyX2RldiIsCiAgICAgICJlbWFpbCI6ICJ1c2VyX2RldkBmb28uY29tIiwKICAgICAgImlzX29yZ19hZG1pbiI6IHRydWUsCiAgICAgICJhY2Nlc3MiOiB7fQogICAgfQogIH0sCiAgIm9yZ19pZCI6ICIwMDAwMDEiLAogICJlbnRpdGxlbWVudHMiOiB7CiAgICAiY29zdF9tYW5hZ2VtZW50IjogewogICAgICAiaXNfZW50aXRsZWQiOiB0cnVlCiAgICB9CiAgfQp9Cg==", "account": "0000001", "org_id": "000001"}' | docker-compose exec -T kafka kafka-console-producer   --bootstrap-server localhost:29092   --topic platform.upload.announce   --property parse.headers=true
```
- Verify: you should see CSV files in the bucket `koku-bucket`