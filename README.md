# คู่มือการพัฒนาแอป iOS รองรับการชำระเงินผ่าน App Store

## บทนำ
เอกสารนี้เป็นคู่มือในการสร้างแอป iOS ด้วย Objective-C ที่รองรับการชำระเงินผ่าน App Store โดยใช้ StoreKit สำหรับจัดการการชำระเงิน และรองรับการซื้อหลายประเภท ได้แก่ ครั้งเดียว, รายสัปดาห์, รายเดือน, และรายปี นอกจากนี้ยังสามารถส่งข้อมูลการทำธุรกรรมไปยังเซิร์ฟเวอร์ผ่าน API และสร้าง JavaScript Interface เพื่อเรียกฟังก์ชันการชำระเงินจาก HTML

## ส่วนประกอบในโปรเจกต์

- **HTML และ JavaScript**: แสดงหน้าชำระเงินและเชื่อมต่อกับ JavaScript Interface เพื่อเรียกใช้การชำระเงินผ่าน Objective-C
- **Objective-C (PaymentManager)**: จัดการการชำระเงินผ่าน StoreKit และส่งข้อมูลการทำธุรกรรมไปยังเซิร์ฟเวอร์

---

## 1. โค้ด HTML และ JavaScript

สร้างไฟล์ `payment.html` เพื่อสร้างหน้าชำระเงินและเชื่อมต่อกับ JavaScript Interface

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Payment Page</title>
</head>
<body>
    <h1>Payment Page</h1>
    <!-- ปุ่มสำหรับการซื้อแบบต่าง ๆ -->
    <button onclick="startPayment('com.yasupada.product.onetime')">One-Time Purchase</button>
    <button onclick="startPayment('com.yasupada.product.weekly')">Weekly Subscription</button>
    <button onclick="startPayment('com.yasupada.product.monthly')">Monthly Subscription</button>
    <button onclick="startPayment('com.yasupada.product.yearly')">Yearly Subscription</button>

    <script>
        function startPayment(productId) {
            // ตรวจสอบว่า JavaScript Interface สามารถเรียกฟังก์ชัน initiatePayment ได้หรือไม่
            if (window.PaymentInterface && typeof window.PaymentInterface.initiatePayment === "function") {
                // เรียกใช้ฟังก์ชันการชำระเงินจาก native interface และส่ง productId ไปด้วย
                window.PaymentInterface.initiatePayment(productId);
            } else {
                alert("Payment not available");
            }
        }
    </script>
</body>
</html>
``` 

### คำอธิบาย
ปุ่มชำระเงิน: แต่ละปุ่มจะเรียก startPayment พร้อม productId ที่สอดคล้องกับประเภทการซื้อ
ฟังก์ชัน startPayment: ตรวจสอบว่า window.PaymentInterface.initiatePayment พร้อมใช้งานและส่ง productId ไปยัง Objective-C

---

## 2. โค้ด Objective-C สำหรับ Payment Interface
สร้างไฟล์ Objective-C ชื่อ PaymentManager.h และ PaymentManager.m เพื่อจัดการการชำระเงิน



PaymentManager.h

```objective-c
#import <Foundation/Foundation.h>
#import <StoreKit/StoreKit.h>
#import <WebKit/WebKit.h>

@interface PaymentManager : NSObject <SKPaymentTransactionObserver, SKProductsRequestDelegate>
+ (instancetype)sharedManager;
- (void)initiatePayment:(NSString *)productIdentifier;
@end
```

PaymentManager.m
```objective-c
#import "PaymentManager.h"

@implementation PaymentManager

+ (instancetype)sharedManager {
    static PaymentManager *sharedManager = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedManager = [[self alloc] init];
    });
    return sharedManager;
}

- (void)initiatePayment:(NSString *)productIdentifier {
    if ([SKPaymentQueue canMakePayments]) {
        SKProductsRequest *request = [[SKProductsRequest alloc] initWithProductIdentifiers:[NSSet setWithObject:productIdentifier]];
        request.delegate = self;
        [request start];
    } else {
        NSLog(@"In-app purchases are disabled.");
    }
}

// SKProductsRequestDelegate method
- (void)productsRequest:(SKProductsRequest *)request didReceiveResponse:(SKProductsResponse *)response {
    SKProduct *product = response.products.firstObject;
    if (product) {
        SKPayment *payment = [SKPayment paymentWithProduct:product];
        [[SKPaymentQueue defaultQueue] addPayment:payment];
    } else {
        NSLog(@"Product not found.");
    }
}

// SKPaymentTransactionObserver method
- (void)paymentQueue:(SKPaymentQueue *)queue updatedTransactions:(NSArray<SKPaymentTransaction *> *)transactions {
    for (SKPaymentTransaction *transaction in transactions) {
        switch (transaction.transactionState) {
            case SKPaymentTransactionStatePurchased:
                [self handleTransaction:transaction];
                [[SKPaymentQueue defaultQueue] finishTransaction:transaction];
                break;
            case SKPaymentTransactionStateFailed:
                NSLog(@"Transaction failed");
                [[SKPaymentQueue defaultQueue] finishTransaction:transaction];
                break;
            default:
                break;
        }
    }
}

- (void)handleTransaction:(SKPaymentTransaction *)transaction {
    NSString *productID = transaction.payment.productIdentifier;
    NSString *transactionID = transaction.transactionIdentifier;
    NSURL *receiptURL = [[NSBundle mainBundle] appStoreReceiptURL];
    NSData *receiptData = [NSData dataWithContentsOfURL:receiptURL];
    NSString *receiptString = [receiptData base64EncodedStringWithOptions:0];

    if (receiptString) {
        NSDictionary *transactionData = @{
            @"productID": productID,
            @"transactionID": transactionID,
            @"receipt": receiptString
        };
        // ส่งข้อมูล transaction ไปยังเซิร์ฟเวอร์
        [self sendTransactionDataToServer:transactionData];
    } else {
        NSLog(@"Receipt data not found.");
    }
}

- (void)sendTransactionDataToServer:(NSDictionary *)transactionData {
    // ส่งข้อมูลการชำระเงินไปยังเซิร์ฟเวอร์
    NSURL *url = [NSURL URLWithString:@"https://yourserver.com/api/payment"];
    NSMutableURLRequest *request = [NSMutableURLRequest requestWithURL:url];
    request.HTTPMethod = @"POST";
    request.HTTPBody = [NSJSONSerialization dataWithJSONObject:transactionData options:0 error:nil];
    [request setValue:@"application/json" forHTTPHeaderField:@"Content-Type"];
    
    [[[NSURLSession sharedSession] dataTaskWithRequest:request completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
        if (error) {
            NSLog(@"Error sending transaction data: %@", error);
        } else {
            NSLog(@"Transaction data sent successfully");
        }
    }] resume];
}

@end
```

## คำอธิบาย
- **initiatePayment:** เริ่มการชำระเงินโดยการสร้างคำขอผลิตภัณฑ์และตรวจสอบว่าผู้ใช้สามารถซื้อได้
- **productsRequest:didReceiveResponse::** เมื่อได้รับข้อมูลผลิตภัณฑ์ จะสร้าง SKPayment และเพิ่มไปใน SKPaymentQueue
- **paymentQueue:updatedTransactions::** ตรวจสอบสถานะของการทำธุรกรรม
- **handleTransaction:** ดึงข้อมูลการทำธุรกรรม (productID, transactionID, และ receiptData) และส่งไปยังเซิร์ฟเวอร์
- **sendTransactionDataToServer:** ส่งข้อมูลการทำธุรกรรมไปยังเซิร์ฟเวอร์โดยใช้ POST ในรูปแบบ JSON

---

## 3. ส่งข้อมูลไปยังเซิร์ฟเวอร์ด้วย PHP
สร้างไฟล์ PHP ชื่อ payment.php บนเซิร์ฟเวอร์เพื่อรับข้อมูลการทำธุรกรรมที่ส่งมาจาก iOS และตรวจสอบใบเสร็จการชำระเงิน

## payment.php
```php
<?php
$data = json_decode(file_get_contents('php://input'), true);

if (isset($data['receipt']) && isset($data['transactionID']) && isset($data['productID'])) {
    $receipt = $data['receipt'];
    $transactionID = $data['transactionID'];
    $productID = $data['productID'];

    // ตรวจสอบใบเสร็จการชำระเงินจาก App Store
    $url = 'https://sandbox.itunes.apple.com/verifyReceipt'; // ใช้ URL นี้สำหรับทดสอบ, สำหรับการใช้งานจริงให้เปลี่ยนเป็น production URL
    $postData = json_encode(['receipt-data' => $receipt]);

    $ch = curl_init($url);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_POST, true);
    curl_setopt($ch, CURLOPT_POSTFIELDS, $postData);
    curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/json']);

    $response = curl_exec($ch);
    curl_close($ch);

    $response = json_decode($response, true);
    
    if ($response['status'] == 0) {
        // ใบเสร็จถูกต้อง
        echo json_encode(['status' => 'success', 'message' => 'Payment verified']);
    } else {
        // ใบเสร็จไม่ถูกต้อง
        echo json_encode(['status' => 'error', 'message' => 'Invalid receipt']);
    }
} else {
    echo json_encode(['status' => 'error', 'message' => 'Invalid request']);
}
?>
```

### คำอธิบาย
- **รับข้อมูล:** รับ receipt, transactionID, และ productID
- **ตรวจสอบใบเสร็จ:** ส่ง receipt ไปยัง Apple เพื่อยืนยันการชำระเงิน
- **การตอบกลับ:** ยืนยันการชำระเงินหรือแจ้งว่าใบเสร็จไม่ถูกต้อง

---

# การทดสอบการชำระเงินใน iOS App Store บน Emulator และ TestFlight

## 1. การทดสอบบน Emulator

**ข้อควรสังเกต**:
- StoreKit ไม่รองรับการทดสอบการชำระเงินบน iOS Simulator (Emulator)
- การทำงานที่เกี่ยวข้องกับ `SKPaymentQueue` และ `SKPayment` จะไม่สามารถทำได้บน Emulator

**แนวทาง**:
- ใช้ **อุปกรณ์จริง** เพื่อทดสอบการชำระเงินด้วย StoreKit
- หากต้องการทดสอบฟังก์ชันการทำงานอื่น ๆ บน Emulator ที่ไม่เกี่ยวข้องกับการชำระเงิน สามารถใช้ Emulator ได้ แต่ไม่สามารถทดสอบการทำธุรกรรมได้

## 2. การทดสอบบน TestFlight

TestFlight เป็นวิธีการทดสอบการทำธุรกรรมของ App Store บน **อุปกรณ์จริง** ซึ่งรองรับทุกประเภทการซื้อ (One-Time Purchase, Subscription แบบรายสัปดาห์, รายเดือน, และรายปี)

### ขั้นตอนการทดสอบบน TestFlight

1. **สมัครบัญชีนักพัฒนา Apple (Apple Developer Account)**: จำเป็นต้องมีบัญชีนักพัฒนาเพื่อใช้งาน TestFlight และเผยแพร่แอป

2. **สร้างแอปและการซื้อภายในแอป (In-App Purchase)**:
   - ลงชื่อเข้าใช้ App Store Connect
   - สร้างโปรเจกต์และกำหนดรายละเอียดต่าง ๆ ของแอป
   - สร้าง `In-App Purchases` ภายใต้การตั้งค่าแอป โดยกำหนด `Product ID` ที่ตรงกับในแอป

3. **การทดสอบการซื้อใน TestFlight**:
   - อัปโหลดแอปไปยัง TestFlight ผ่าน Xcode
   - เชิญผู้ใช้ทดสอบด้วยอีเมลผ่าน TestFlight
   - เมื่อทำการทดสอบ จะใช้บัญชีผู้ทดสอบของ TestFlight เพื่อซื้อ In-App Purchase แบบ Sandbox โดยไม่เสียค่าใช้จ่ายจริง ๆ
   - ตรวจสอบว่าใบเสร็จ (`receipt`) และข้อมูลการทำธุรกรรมสามารถตรวจสอบและใช้งานได้ตามที่ออกแบบ

4. **ตรวจสอบการยืนยันการชำระเงิน**:
   - ใช้เซิร์ฟเวอร์เพื่อยืนยันการชำระเงินตามที่ตั้งค่าไว้ในไฟล์ `payment.php`
   - ใช้ iTunes Connect ในการตรวจสอบสถานะการชำระเงินแบบ Sandbox

---

## ปัญหาที่อาจเกิดขึ้นและการแก้ไข

### 1. ไม่สามารถรับข้อมูล `receipt`
   - **ปัญหา**: `receipt` อาจไม่สามารถดึงข้อมูลได้หรือมีข้อมูลไม่ครบ
   - **แนวทางแก้ไข**: ตรวจสอบการเรียก `appStoreReceiptURL` และตรวจสอบว่าใบเสร็จมีอยู่จริงใน `NSBundle.mainBundle`

### 2. Product ID ไม่ตรงกับการตั้งค่าใน App Store Connect
   - **ปัญหา**: `Product ID` ที่ใช้ในโค้ดไม่ตรงกับ `Product ID` ที่ตั้งค่าไว้ใน App Store Connect
   - **แนวทางแก้ไข**: ตรวจสอบว่า `Product ID` ใน StoreKit และ App Store Connect ตรงกัน

### 3. Status ของใบเสร็จไม่เท่ากับ 0
   - **ปัญหา**: เมื่อส่ง `receipt` ไปยังเซิร์ฟเวอร์ Apple แล้วสถานะ (`status`) ของใบเสร็จไม่เท่ากับ 0 ซึ่งหมายความว่าใบเสร็จไม่ถูกต้อง
   - **แนวทางแก้ไข**: ตรวจสอบ URL ของ `verifyReceipt` ว่าใช้ sandbox หรือ production URL ให้ถูกต้องตามสภาพแวดล้อมที่ทดสอบ

### 4. Transaction ซ้ำ
   - **ปัญหา**: ในการทดสอบ subscription อาจเกิด `Transaction` ที่ซ้ำกันเมื่อทดลองซื้อซ้ำ
   - **แนวทางแก้ไข**: ใช้ `transaction.transactionIdentifier` เพื่อตรวจสอบว่าเคยดำเนินการไปแล้วหรือไม่

### 5. การตรวจสอบใบเสร็จไม่ผ่าน
   - **ปัญหา**: ข้อมูลการยืนยันจาก Apple อาจระบุว่าใบเสร็จไม่ถูกต้อง (status ≠ 0)
   - **แนวทางแก้ไข**: 
     - ตรวจสอบว่าการส่งข้อมูล `receipt-data` ไปยัง Apple ถูกต้องหรือไม่
     - ตรวจสอบการตั้งค่าผลิตภัณฑ์ใน App Store Connect ว่ามีสถานะ `Ready to Submit` หรือไม่

---

## สรุป

การทดสอบการชำระเงินควรทำบนอุปกรณ์จริงโดยใช้ TestFlight เพื่อให้ได้ข้อมูลการทดสอบที่สมบูรณ์และเสถียร การแก้ไขปัญหาและการตรวจสอบข้อมูล `receipt` ควรทดสอบอย่างละเอียดเพื่อป้องกันข้อผิดพลาดในการทำธุรกรรมในแอปที่เผยแพร่
