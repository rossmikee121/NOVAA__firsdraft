# PAYMENT PROCESSING GUIDE

**For**: Backend developers  
**Version**: 1.0  
**Date**: January 20, 2026  
**Critical**: Handle payments carefully - this is real money

---

## 💳 OVERVIEW

NOVAA integrates Razorpay for:
- Fee collection from students
- Multi-currency support (future)
- Payment tracking
- Automated receipts

**Key Requirements**:
- Prevent duplicate charges (idempotency keys)
- Calculate GST correctly (18% on taxable fees)
- Track payment status for compliance
- Generate audit trail for each transaction

---

## 💰 FEE STRUCTURE & GST CALCULATION

### Fee Types

```javascript
// Admission Fee: 500 INR (no tax)
// Tuition: Variable (no tax - educational)
// Lab Fee: Variable (18% GST)
// Sports Fee: Variable (18% GST)
// Library Fee: 100 INR (18% GST)
// Transport Fee: Variable (5% GST)

// Example:
// Tuition: 50,000 (tax: 0)
// Lab: 5,000 (tax: 900)
// Sports: 2,000 (tax: 360)
// Total: 57,000 + 1,260 (GST) = 58,260 INR
```

### GST Calculation Function

```javascript
// utils/gst.js
class GSTCalculator {
  static RATES = {
    TUITION: 0.00,        // Educational - no tax
    LAB: 0.18,            // 18% GST
    SPORTS: 0.18,         // 18% GST
    LIBRARY: 0.18,        // 18% GST
    TRANSPORT: 0.05,      // 5% GST
    ADMISSION: 0.00       // Service - may vary
  };

  static calculateGST(feeType, amount) {
    const rate = this.RATES[feeType];
    if (!rate) {
      throw new Error(`Unknown fee type: ${feeType}`);
    }
    return Math.round(amount * rate);
  }

  static calculateTotal(fees) {
    // fees = { tuition: 50000, lab: 5000, sports: 2000 }
    let subtotal = 0;
    let totalGST = 0;

    for (const [feeType, amount] of Object.entries(fees)) {
      const gst = this.calculateGST(feeType.toUpperCase(), amount);
      subtotal += amount;
      totalGST += gst;
    }

    return {
      subtotal,
      totalGST,
      total: subtotal + totalGST
    };
  }
}

// Usage
const breakup = GSTCalculator.calculateTotal({
  tuition: 50000,
  lab: 5000,
  sports: 2000
});

console.log(breakup);
// {
//   subtotal: 57000,
//   totalGST: 1260,
//   total: 58260
// }
```

---

## 🔌 RAZORPAY INTEGRATION

### Setup

```javascript
// config/razorpay.js
const Razorpay = require('razorpay');

const razorpay = new Razorpay({
  key_id: process.env.RAZORPAY_KEY_ID,
  key_secret: process.env.RAZORPAY_KEY_SECRET
});

module.exports = razorpay;

// .env
RAZORPAY_KEY_ID=rzp_test_xxxxxxxxxxxxxxxx
RAZORPAY_KEY_SECRET=xxxxxxxxxxxxxxxxxxxxxxxx
```

### Order Creation (Backend)

```javascript
// controllers/paymentController.js
const createOrder = async (req, res) => {
  try {
    const { studentId, admissionId, amount, description } = req.body;
    const collegeId = req.user.collegeId;

    // 1. Validate amount
    if (amount <= 0) {
      return res.status(400).json({ error: 'Invalid amount' });
    }

    // 2. Verify student is from same college
    const student = await Student.findOne({
      _id: studentId,
      collegeId
    });

    if (!student) {
      return res.status(404).json({ error: 'Student not found' });
    }

    // 3. Create Razorpay order
    const order = await razorpay.orders.create({
      amount: amount * 100,        // Razorpay expects paise
      currency: 'INR',
      receipt: `admission-${admissionId}`,
      notes: {
        studentId,
        admissionId,
        collegeId: collegeId.toString()
      }
    });

    // 4. Save order to database
    const transaction = await Transaction.create({
      studentId,
      admissionId,
      collegeId,
      razorpayOrderId: order.id,
      amount,
      status: 'ORDER_CREATED',
      orderDetails: {
        amount: order.amount,
        currency: order.currency,
        receipt: order.receipt
      }
    });

    // 5. Return to frontend
    res.status(201).json({
      orderId: order.id,
      amount: order.amount,
      currency: order.currency,
      transactionId: transaction._id
    });

  } catch (error) {
    console.error('Order creation error:', error);
    res.status(500).json({ error: 'Failed to create order' });
  }
};

router.post('/orders', authenticate, createOrder);
```

### Payment Verification (Backend)

```javascript
// controllers/paymentController.js
const verifyPayment = async (req, res) => {
  try {
    const { razorpayOrderId, razorpayPaymentId, razorpaySignature } = req.body;
    const collegeId = req.user.collegeId;

    // 1. Verify signature (critical for security)
    const shasum = crypto
      .createHmac('sha256', process.env.RAZORPAY_KEY_SECRET)
      .update(`${razorpayOrderId}|${razorpayPaymentId}`)
      .digest('hex');

    if (shasum !== razorpaySignature) {
      return res.status(400).json({ error: 'Invalid signature' });
    }

    // 2. Find transaction
    const transaction = await Transaction.findOne({
      razorpayOrderId,
      collegeId
    });

    if (!transaction) {
      return res.status(404).json({ error: 'Transaction not found' });
    }

    // 3. Check if already processed (prevent duplicates)
    if (transaction.status === 'PAYMENT_VERIFIED') {
      return res.status(200).json({
        message: 'Payment already verified',
        transaction
      });
    }

    // 4. Fetch payment details from Razorpay
    const payment = await razorpay.payments.fetch(razorpayPaymentId);

    if (payment.status !== 'captured') {
      return res.status(400).json({
        error: 'Payment not captured'
      });
    }

    // 5. Update transaction
    transaction.status = 'PAYMENT_VERIFIED';
    transaction.razorpayPaymentId = razorpayPaymentId;
    transaction.razorpaySignature = razorpaySignature;
    transaction.verifiedAt = new Date();
    transaction.paymentDetails = {
      amount: payment.amount,
      currency: payment.currency,
      method: payment.method,
      card: payment.card_id ? payment.card : null
    };

    await transaction.save();

    // 6. Update admission status
    await Admission.findByIdAndUpdate(
      transaction.admissionId,
      {
        status: 'PAYMENT_RECEIVED',
        paymentVerifiedAt: new Date()
      }
    );

    // 7. Generate receipt
    const receipt = generateReceipt(transaction, payment);

    // 8. Send receipt email
    await emailService.sendReceipt(transaction.studentId, receipt);

    // 9. Return success
    res.status(200).json({
      message: 'Payment verified successfully',
      transaction,
      receipt
    });

  } catch (error) {
    console.error('Verification error:', error);
    res.status(500).json({ error: 'Verification failed' });
  }
};

router.post('/verify', authenticate, verifyPayment);
```

---

## 🛡️ IDEMPOTENCY & DUPLICATE PREVENTION

### Idempotency Key Pattern

```javascript
// models/Transaction.js
const transactionSchema = new Schema({
  studentId: { type: Schema.Types.ObjectId, ref: 'Student', required: true },
  admissionId: { type: Schema.Types.ObjectId, ref: 'Admission', required: true },
  collegeId: { type: Schema.Types.ObjectId, ref: 'College', required: true },
  amount: { type: Number, required: true },
  
  // Idempotency
  idempotencyKey: {
    type: String,
    unique: true,
    required: true,
    index: true
    // Format: `college-${collegeId}-student-${studentId}-admission-${admissionId}`
  },

  razorpayOrderId: String,
  razorpayPaymentId: String,
  razorpaySignature: String,

  status: {
    type: String,
    enum: ['PENDING', 'ORDER_CREATED', 'PAYMENT_VERIFIED', 'FAILED'],
    default: 'PENDING'
  },

  createdAt: { type: Date, default: Date.now },
  verifiedAt: Date,
  failedAt: Date
});

// Unique index on idempotency key
transactionSchema.index({ idempotencyKey: 1 }, { unique: true });
```

### Creating Order with Idempotency

```javascript
const createOrderIdempotent = async (req, res) => {
  try {
    const { studentId, admissionId, amount } = req.body;
    const collegeId = req.user.collegeId;

    // 1. Generate idempotency key
    const idempotencyKey = `college-${collegeId}-student-${studentId}-admission-${admissionId}`;

    // 2. Check if order already exists
    let transaction = await Transaction.findOne({ idempotencyKey });

    if (transaction) {
      // Order already created, return existing
      return res.status(200).json({
        message: 'Using existing order',
        orderId: transaction.razorpayOrderId,
        amount: transaction.amount,
        transactionId: transaction._id
      });
    }

    // 3. Create new order
    const razorpayOrder = await razorpay.orders.create({
      amount: amount * 100,
      currency: 'INR',
      receipt: idempotencyKey
    });

    // 4. Save transaction with idempotency key
    transaction = await Transaction.create({
      studentId,
      admissionId,
      collegeId,
      idempotencyKey,
      razorpayOrderId: razorpayOrder.id,
      amount,
      status: 'ORDER_CREATED'
    });

    res.status(201).json({
      orderId: razorpayOrder.id,
      amount: razorpayOrder.amount,
      transactionId: transaction._id
    });

  } catch (error) {
    if (error.code === 11000) {
      // Duplicate key - return existing transaction
      const transaction = await Transaction.findOne({
        idempotencyKey: `college-${req.user.collegeId}-student-${req.body.studentId}-admission-${req.body.admissionId}`
      });
      return res.status(200).json({
        message: 'Using existing order',
        orderId: transaction.razorpayOrderId
      });
    }

    res.status(500).json({ error: 'Failed to create order' });
  }
};
```

---

## 📲 RAZORPAY WEBHOOKS

### Webhook Setup

```javascript
// Razorpay Dashboard → Settings → Webhooks
// URL: https://yourdomain.com/api/payments/webhook
// Events:
//   - payment.authorized
//   - payment.failed
//   - payment.captured
//   - refund.created
```

### Webhook Handler

```javascript
// controllers/webhookController.js
const handleWebhook = async (req, res) => {
  try {
    // 1. Verify webhook signature
    const crypto = require('crypto');
    const signature = req.get('X-Razorpay-Signature');

    const body = JSON.stringify(req.body);
    const expectedSignature = crypto
      .createHmac('sha256', process.env.RAZORPAY_WEBHOOK_SECRET)
      .update(body)
      .digest('hex');

    if (signature !== expectedSignature) {
      return res.status(400).json({ error: 'Invalid webhook' });
    }

    // 2. Handle different events
    const { event, payload } = req.body;

    switch (event) {
      case 'payment.authorized':
        await handlePaymentAuthorized(payload.payment);
        break;

      case 'payment.captured':
        await handlePaymentCaptured(payload.payment);
        break;

      case 'payment.failed':
        await handlePaymentFailed(payload.payment);
        break;

      case 'refund.created':
        await handleRefundCreated(payload.refund);
        break;

      default:
        console.log('Unknown event:', event);
    }

    // 3. Always return 200 to acknowledge receipt
    res.status(200).json({ status: 'ok' });

  } catch (error) {
    console.error('Webhook error:', error);
    res.status(200).json({ status: 'ok' }); // Still return 200
  }
};

const handlePaymentCaptured = async (payment) => {
  const transaction = await Transaction.findOne({
    razorpayPaymentId: payment.id
  });

  if (!transaction) return;

  transaction.status = 'PAYMENT_VERIFIED';
  transaction.verifiedAt = new Date();
  await transaction.save();

  // Update admission
  await Admission.findByIdAndUpdate(transaction.admissionId, {
    status: 'PAYMENT_RECEIVED',
    paymentVerifiedAt: new Date()
  });

  // Send email
  await emailService.sendReceipt(transaction.studentId, transaction);
};

const handlePaymentFailed = async (payment) => {
  const transaction = await Transaction.findOne({
    razorpayPaymentId: payment.id
  });

  if (!transaction) return;

  transaction.status = 'FAILED';
  transaction.failedAt = new Date();
  transaction.failureReason = payment.description;
  await transaction.save();

  // Notify student
  await emailService.sendPaymentFailed(
    transaction.studentId,
    transaction.amount
  );
};

router.post('/webhook', handleWebhook);
```

---

## 💵 REFUND HANDLING

### Processing Refunds

```javascript
// controllers/refundController.js
const processRefund = async (req, res) => {
  try {
    const { transactionId, reason } = req.body;
    const collegeId = req.user.collegeId;

    // 1. Only admins can process refunds
    if (req.user.role !== 'ADMIN') {
      return res.status(403).json({ error: 'Permission denied' });
    }

    // 2. Find transaction
    const transaction = await Transaction.findOne({
      _id: transactionId,
      collegeId
    });

    if (!transaction) {
      return res.status(404).json({ error: 'Transaction not found' });
    }

    // 3. Check if already refunded
    if (transaction.status === 'REFUNDED') {
      return res.status(400).json({ error: 'Already refunded' });
    }

    // 4. Process refund with Razorpay
    const refund = await razorpay.payments.refund(
      transaction.razorpayPaymentId,
      {
        amount: transaction.amount * 100,  // In paise
        notes: {
          reason,
          processedBy: req.user.userId
        }
      }
    );

    // 5. Update transaction
    transaction.status = 'REFUNDED';
    transaction.refundId = refund.id;
    transaction.refundReason = reason;
    transaction.refundProcessedAt = new Date();
    await transaction.save();

    // 6. Update admission
    await Admission.findByIdAndUpdate(transaction.admissionId, {
      status: 'REFUNDED'
    });

    // 7. Notify student
    await emailService.sendRefundConfirmation(
      transaction.studentId,
      transaction.amount,
      reason
    );

    res.json({
      message: 'Refund processed',
      refundId: refund.id,
      transaction
    });

  } catch (error) {
    console.error('Refund error:', error);
    res.status(500).json({ error: 'Refund failed' });
  }
};

router.post('/refund', authenticate, processRefund);
```

---

## 📊 PAYMENT RETRY LOGIC

### Failed Payment Recovery

```javascript
// services/paymentRetryService.js
class PaymentRetryService {
  // Retry failed payments automatically
  static async retryFailedPayments() {
    try {
      const failedTransactions = await Transaction.find({
        status: 'FAILED',
        retryCount: { $lt: 3 },
        failedAt: { $gt: new Date(Date.now() - 24 * 60 * 60 * 1000) } // Last 24h
      });

      for (const transaction of failedTransactions) {
        await this.retryPayment(transaction);
      }

    } catch (error) {
      console.error('Retry error:', error);
    }
  }

  static async retryPayment(transaction) {
    try {
      // 1. Fetch latest payment status
      const payments = await razorpay.orders.fetchPayments(
        transaction.razorpayOrderId
      );

      if (payments.items.length === 0) {
        // No payment attempted, notify student to retry
        await emailService.sendPaymentRetryRequest(
          transaction.studentId,
          transaction.admissionId
        );
      }

      // 2. Increment retry count
      transaction.retryCount = (transaction.retryCount || 0) + 1;
      await transaction.save();

    } catch (error) {
      console.error('Payment retry failed:', error);
    }
  }

  // Manual retry endpoint
  static async manualRetry(transactionId, collegeId) {
    const transaction = await Transaction.findOne({
      _id: transactionId,
      collegeId
    });

    if (!transaction) {
      throw new Error('Transaction not found');
    }

    // Create new order
    const newOrder = await razorpay.orders.create({
      amount: transaction.amount * 100,
      currency: 'INR',
      receipt: `${transaction.admissionId}-retry-${Date.now()}`
    });

    return {
      orderId: newOrder.id,
      amount: newOrder.amount
    };
  }
}

// Scheduler (run every hour)
const schedule = require('node-schedule');
schedule.scheduleJob('0 * * * *', () => {
  PaymentRetryService.retryFailedPayments();
});
```

---

## 📱 FRONTEND PAYMENT INTEGRATION

### React Component

```javascript
// components/PaymentForm.js
import React, { useState } from 'react';
import paymentService from '../services/paymentService';

const PaymentForm = ({ admissionId, studentId, amount }) => {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState('');

  const handlePayment = async () => {
    try {
      setLoading(true);
      setError('');

      // 1. Create order
      const { orderId, transactionId } = await paymentService.createOrder({
        studentId,
        admissionId,
        amount
      });

      // 2. Initialize Razorpay
      const options = {
        key: process.env.REACT_APP_RAZORPAY_KEY_ID,
        amount: amount * 100,
        currency: 'INR',
        order_id: orderId,
        handler: async (response) => {
          // Payment successful
          try {
            await paymentService.verifyPayment({
              razorpayOrderId: orderId,
              razorpayPaymentId: response.razorpay_payment_id,
              razorpaySignature: response.razorpay_signature,
              transactionId
            });

            alert('Payment successful!');
          } catch (err) {
            setError('Payment verification failed');
          }
        },
        prefill: {
          name: studentId,
          email: 'student@example.com',
          contact: '9999999999'
        },
        theme: {
          color: '#3399cc'
        },
        modal: {
          ondismiss: () => {
            setError('Payment cancelled');
          }
        }
      };

      const rzp = new window.Razorpay(options);
      rzp.open();

    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div>
      <h3>Pay ₹{amount}</h3>
      {error && <div className="error">{error}</div>}
      <button 
        onClick={handlePayment} 
        disabled={loading}
      >
        {loading ? 'Processing...' : 'Pay Now'}
      </button>
    </div>
  );
};

export default PaymentForm;
```

### Payment Service

```javascript
// services/paymentService.js
class PaymentService {
  static async createOrder(data) {
    const response = await apiClient.post('/api/payments/orders', data);
    return response.data;
  }

  static async verifyPayment(data) {
    const response = await apiClient.post('/api/payments/verify', data);
    return response.data;
  }

  static async getTransactionStatus(transactionId) {
    const response = await apiClient.get(
      `/api/payments/transactions/${transactionId}`
    );
    return response.data;
  }

  static async retryPayment(transactionId) {
    const response = await apiClient.post(
      `/api/payments/retry/${transactionId}`
    );
    return response.data;
  }
}

export default PaymentService;
```

---

## 📋 PAYMENT CHECKLIST

- [ ] GST calculated correctly for each fee type
- [ ] Idempotency key implemented to prevent duplicates
- [ ] Razorpay signature verified before payment confirmation
- [ ] Transaction logged in database before updating status
- [ ] Receipt generated and sent to student
- [ ] Multi-tenancy enforced (collegeId checked)
- [ ] Refunds tracked separately
- [ ] Webhook handler verifies signature
- [ ] Failed payment retry logic implemented
- [ ] Error messages don't expose sensitive info
- [ ] All amounts in paise for Razorpay
- [ ] Payment status updates sent to admission module
- [ ] Audit trail maintained for compliance
- [ ] Timeout handling for payment verification
- [ ] Rate limiting on payment endpoints

---

## 📊 NEXT DOCUMENTS

1. ✅ **01_MERN_STACK_OVERVIEW.md**
2. ✅ **02_DEVELOPMENT_ENVIRONMENT_SETUP.md**
3. ✅ **03_MODULES_ARCHITECTURE.md**
4. ✅ **04_MODULE_INTERCONNECTIONS.md**
5. ✅ **05_DATABASE_DEVELOPER_GUIDE.md**
6. ✅ **06_API_DEVELOPMENT_GUIDE.md**
7. ✅ **07_FRONTEND_DEVELOPMENT_GUIDE.md**
8. ✅ **08_CODE_STANDARDS_CONVENTIONS.md**
9. ✅ **09_AUTHENTICATION_SECURITY_GUIDE.md**
10. ✅ **10_PAYMENT_PROCESSING_GUIDE.md** (You are here)
11. **11_TESTING_DEVELOPER_GUIDE.md**
12. **12_DEBUGGING_TROUBLESHOOTING.md**

---

**Previous**: [09_AUTHENTICATION_SECURITY_GUIDE.md](09_AUTHENTICATION_SECURITY_GUIDE.md)  
**Next Document**: [11_TESTING_DEVELOPER_GUIDE.md](11_TESTING_DEVELOPER_GUIDE.md)

