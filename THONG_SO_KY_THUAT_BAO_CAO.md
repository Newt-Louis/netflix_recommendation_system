# Các thông số kỹ thuật cần ghi trong báo cáo

## Môi trường thực nghiệm

| Thành phần | Thông số chốt cho đồ án |
|---|---|
| Hệ điều hành máy host | Windows 11 |
| Cơ chế chạy container | Docker Desktop trên Windows 11, Linux containers |
| Tổng CPU cấp cho môi trường giả lập | 6 CPU cores |
| Tổng RAM cấp cho môi trường giả lập | 10GB |
| Tổng dung lượng đĩa cấp cho Docker/dataset/outputs | 40GB |
| GPU | 1 NVIDIA GPU, cấp cho container Notebook/TensorFlow |
| Dataset | MovieLens 25M |
| Số container | 5 container |
| Database ngoài | Không sử dụng |
| Kafka / streaming broker | Không sử dụng |

## Phiên bản phần mềm

| Thành phần | Phiên bản / image sử dụng | Vai trò |
|---|---|---|
| Hadoop/HDFS | apache/hadoop:3.4.3 | Lưu trữ dữ liệu MovieLens 25M và dữ liệu trung gian trên HDFS |
| Spark | bitnami/spark:3.5.8 | Xử lý dữ liệu lớn, EDA, feature engineering, train ALS bằng Spark MLlib |
| PySpark | pyspark==3.5.8 | Driver Python trong Jupyter Notebook kết nối Spark cluster |
| TensorFlow | tensorflow/tensorflow:2.21.0-gpu-jupyter | Huấn luyện mô hình Neural Collaborative Filtering có GPU |
| Java | OpenJDK 17 | Runtime cho PySpark/Spark client trong notebook |
| Python | Python đi kèm TensorFlow image | Viết notebook, tiền xử lý bổ trợ, huấn luyện NCF |
| JupyterLab | Đi kèm TensorFlow Jupyter image | Môi trường thực nghiệm và ghi nhận kết quả |

## Phân bổ tài nguyên container

| Container | Image | CPU | RAM | GPU | Volume chính | Port |
|---|---|---:|---:|---|---|---|
| namenode | apache/hadoop:3.4.3 | 0.50 core | 768MB | Không | namenode_data | 9870, 8020 |
| datanode | apache/hadoop:3.4.3 | 1.00 core | 1536MB | Không | datanode_data | 9864 |
| spark-master | bitnami/spark:3.5.8 | 0.50 core | 768MB | Không | spark_master_work | 8080, 7077 |
| spark-worker | bitnami/spark:3.5.8 | 2.50 cores | 4GB | Không | spark_worker_work, ./data, ./outputs | 8081 |
| notebook-tensorflow | tensorflow/tensorflow:2.21.0-gpu-jupyter + PySpark 3.5.8 | 1.50 cores | 3GB | 1 GPU | ./data, ./notebooks, ./src, ./outputs, ./models | 8888, 4040, 8501 |
| Tổng | 5 containers | 6.00 cores | 10GB | 1 GPU | Docker named volumes + bind mounts | - |

## Vai trò từng container

- `namenode`: quản lý metadata của HDFS, cung cấp HDFS Web UI tại cổng 9870 và HDFS RPC tại cổng 8020.
- `datanode`: lưu trữ block dữ liệu thực tế của HDFS; replication được đặt bằng 1 để tiết kiệm dung lượng trong môi trường giả lập một DataNode.
- `spark-master`: quản lý Spark standalone cluster và phân phối ứng dụng Spark cho worker.
- `spark-worker`: thực thi các tác vụ Spark, bao gồm đọc dữ liệu MovieLens từ HDFS, EDA, tiền xử lý, tạo đặc trưng và huấn luyện ALS.
- `notebook-tensorflow`: môi trường JupyterLab để viết notebook PySpark, gọi Spark cluster, chạy TensorFlow GPU cho mô hình Neural Collaborative Filtering và xuất kết quả khuyến nghị.

## Luồng xử lý dữ liệu

MovieLens 25M CSV → upload vào HDFS → Spark đọc dữ liệu từ HDFS → Spark EDA/tiền xử lý/feature engineering → Spark MLlib huấn luyện ALS → xuất dữ liệu train/test cho TensorFlow → TensorFlow huấn luyện Neural Collaborative Filtering bằng GPU → sinh Top-N recommendation → lưu kết quả vào `outputs/` và model vào `models/`.
