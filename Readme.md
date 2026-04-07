# README

## 1. Metadata của ảnh gốc

Ảnh đầu vào là ảnh vệ tinh đa phổ với các đặc điểm chính:

* Kích thước: **11543 x 8070 pixels**
* Số kênh: **8 bands**
* Kiểu dữ liệu: **UInt16 (16-bit)**
* Giá trị pixel: **0 → 65535 (2¹⁶)**
* Định dạng: **GeoTIFF (có thông tin tọa độ - CRS, transform)**

Các band phổ bao gồm:

* Coastal Blue
* Blue
* Green I
* Green
* Yellow
* Red
* Red Edge
* Near Infrared (NIR)

---

## 2. Bài toán chuyển đổi 16-bit → 8-bit

Ảnh 16-bit có miền giá trị:

* **0 → 2¹⁶ = 65536**

Trong khi đó ảnh 8-bit có miền giá trị:

* **0 → 2⁸ = 256**

Do đó, để hiển thị hoặc sử dụng cho các tác vụ khác thì phải thực hiện biến đổi
```
[0, 65535] → [0, 255]
```

Tuy vậy, ta sẽ gặp các trường hợp như sau khi normalization pixel

* Phân bố pixel không đồng đều
* Có outlier (giá trị cực lớn/cực nhỏ)
* Dynamic range bị lệch ví dụ như việc bị lệch range giữa các band là khác nhau như sau:
	Band 1: Coastal blue (Min: 6970 | Max: 7683) 
	Band 2: blue (Min: 5728 | Max: 6755) 
	Band 3: green_i (Min: 4716 | Max: 5948) 
	Band 4: green (Min: 619 | Max: 12281) 
	Band 5: yellow (Min: 619 | Max: 12281) 
	Band 6: red (Min: 619 | Max: 12281) 
	Band 7: rededge (Min: 619 | Max: 12281) 
	Band 8: Nir (Min: 619 | Max: 12281)

---

## 3. Phân tích histogram và đặc điểm dữ liệu
![Histogram from QGIS](./histogram.jpg)

![Histogram per band](./image.png)
Dựa vào biểu đồ histogram:

* Pixel không phân bố đều trên toàn miền 0 → 12000+
* Mỗi band có một khoảng giá trị tập trung riêng
* Có **đuôi dài (long tail)** → tồn tại outlier
* Phần lớn pixel tập trung ở khoảng thấp

### Boxplot
![Box plot for per band](./bplot.png)
Dựa vào box plot:
* Ta thấy rằng mỗi band có rất nhiều điểm giá trị pixel nằm ngoài khoảng Q1 và Q3 => Có rất nhiều outlier => Càng chứng tỏ sẽ rất nhạy với kiểu min max

### Nhận xét:

1. **Dynamic range thực tế nhỏ hơn lý thuyết**

   * Dù max ~12000 nhưng phần lớn pixel nằm trong khoảng hẹp hơn

2. **Có outlier**

   * Các giá trị cực lớn (cloud, reflection, noise)
   * Nếu dùng min-max sẽ bị kéo giãn sai

3. **Các band không đồng nhất**

   * Mỗi band có phân bố khác nhau → dễ gây lệch màu

---

## 4. Thử nghiệm với 2 phương pháp biến đổi ảnh dựa vào độ lệch chuẩn và percentile

### Vấn đề với Min-Max Scaling cơ bản

```
x' = (x - min) / (max - min)
```

**Nhược điểm lớn:**

* Rất nhạy với outlier (pixel cực lớn/nhỏ)
* Một vài pixel nhiễu làm phần lớn ảnh bị tối
* Mất chi tiết vùng chính vì dynamic range bị giãn quá

---

### Hai phương pháp được sử dụng

#### **1️⃣ Phương pháp Standard Deviation**

Thay vì dùng min/max toàn cục, sử dụng thống kê:

```
mean = trung bình dữ liệu
std = độ lệch chuẩn

---

#### **2️⃣ Phương pháp Percentile**

Giá trị của các band sau khi tính toán như sau:
	Biến đổi thông qua phương thức: percentile
	Band 1: Range [6843.0 - 7975.0]
	Band 2: Range [5577.0 - 7195.0]
	Band 3: Range [4561.0 - 6793.0]
	Band 4: Range [3729.0 - 6383.0]
	Band 5: Range [2503.0 - 5897.0]
	Band 6: Range [1842.0 - 5447.9]
	Band 7: Range [1372.0 - 6053.0]
	Band 8: Range [627.0 - 7095.0]

```
p_low = percentile(0.5%)
p_high = percentile(99.5%)
```

**Tham số:**
- `STRETCH_METHOD = "percentile"`
- `PERCENTILES = (0.5, 99.5)` (cắt bỏ 0.5% pixel ở hai đầu)

## 5. Vấn đề về kích thước ảnh

Ảnh có kích thước:

```
(11543, 8070, 8) => Open image trực tiếp thì sẽ crash note book
```

### => Để đọc được bức ảnh này thì phải đọc theo từng window với các kích thước block nhỏ để tránh gây tràn ram


## 6. Giữ thông tin tọa độ (Georeference) - Lấy đúng thông tin (profile mà thư viện rasterio đọc)

## 7. Ảnh sau khi biến đổi
- Ảnh RGB khi sử dụng phương pháp percentile
[Ảnh RGB](https://drive.google.com/file/d/1xg4L2zrc3KQWLQgJmvqZpEM_cnXaqhN-/view?usp=sharing)

- Ảnh Tif
[Ảnh RGB](https://drive.google.com/file/d/1BtU9qb74vcsULC4Y-nMLYs-TzJ1llP5I/view?usp=sharing)

## 8. Đánh giá ảnh sau khi biến đổi qua từng band:
Kiểm tra chất lượng ảnh sau biến đổi
Band 1: PSNR=21.26 dB | SSIM=0.8517 | Entropy=5.88
Band 2: PSNR=24.06 dB | SSIM=0.7655 | Entropy=5.58
Band 3: PSNR=17.70 dB | SSIM=0.6717 | Entropy=5.52
Band 4: PSNR=13.34 dB | SSIM=0.6154 | Entropy=5.56
Band 5: PSNR=9.35 dB | SSIM=0.3458 | Entropy=4.84
Band 6: PSNR=8.80 dB | SSIM=0.1831 | Entropy=4.57
Band 7: PSNR=7.11 dB | SSIM=0.0802 | Entropy=3.70
Band 8: PSNR=6.81 dB | SSIM=0.0212 | Entropy=2.62

=> Với mục đích để trực quan ảnh (Xem ảnh) thì việc các band RGB giữ được nhiều thông tin là mục đích hướng đến