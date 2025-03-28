## **Ubuntu VPS-এ MongoDB Setup, User Management, এবং Firewall Configuration (Step-by-Step)**
---
এই গাইডে আমি বিস্তারিতভাবে Ubuntu VPS-এ **MongoDB ইনস্টল, ইউজার তৈরি, ডাটাবেস তৈরি, ইউজার ও ডাটাবেস ম্যানেজমেন্ট, Firewall কনফিগারেশন, এবং IP Access Control** সম্পর্কে বলব।

---

## **Step 1: MongoDB Install করা (Locally)**
MongoDB ইনস্টল করার জন্য নিচের কমান্ডগুলো রান করুন:

### **1. MongoDB-এর Official GPG Key Add করুন**
MongoDB-এর প্যাকেজ ভেরিফাই করার জন্য এর GPG key যুক্ত করতে হবে।
```bash
curl -fsSL https://pgp.mongodb.com/server-7.0.asc | sudo gpg --dearmor -o /usr/share/keyrings/mongodb-server-7.0.gpg
```

### **2. MongoDB-এর Official Repository যোগ করুন**
```bash
echo "deb [signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg] https://repo.mongodb.org/apt/ubuntu $(lsb_release -cs)/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
```

### **3. System Update করুন**
```bash
sudo apt update
```

### **4. MongoDB Install করুন**
```bash
sudo apt install -y mongodb-org
```

### **5. MongoDB Service চালু করুন**
```bash
sudo systemctl start mongod
```

### **6. MongoDB সার্ভিস চালু থাকবে নিশ্চিত করুন**
```bash
sudo systemctl enable mongod
```

### **7. MongoDB চালু হয়েছে কিনা চেক করুন**
```bash
sudo systemctl status mongod
```
যদি সব ঠিক থাকে, তাহলে নিচের মতো মেসেজ দেখাবে:
```
● mongod.service - MongoDB Database Server
     Loaded: loaded (/lib/systemd/system/mongod.service; enabled; vendor preset: enabled)
     Active: active (running) since...
```
এই স্ট্যাটাস পেতে **q চাপলে** টার্মিনাল থেকে বেরিয়ে আসবে।

---

## **Step 2: MongoDB Shell-এ ঢোকা**
MongoDB Shell চালু করতে:
```bash
mongosh
```
এতে MongoDB-এর ইন্টারেক্টিভ কনসোল চালু হবে।

---

## **Step 3: নতুন Database তৈরি করা**
MongoDB তে নতুন ডাটাবেস তৈরি করতে:
```javascript
use myDatabase
```
এটি `myDatabase` নামে একটি নতুন ডাটাবেস তৈরি করবে। তবে এটি তখনই সংরক্ষিত হবে যখন আপনি এতে ডাটা যুক্ত করবেন।

ডাটাবেসে একটি কালেকশন (টেবিলের মতো) তৈরি করুন:
```javascript
db.myCollection.insertOne({ name: "Test Data" })
```
ডাটাবেস লিস্ট দেখতে:
```javascript
show dbs
```
---

## **Step 4: নতুন User তৈরি করা ও ডাটাবেসের সাথে সংযুক্ত করা**
```javascript
use admin
db.createUser({
  user: "myUser",
  pwd: "myPassword",
  roles: [{ role: "readWrite", db: "myDatabase" }]
});
```
**ইউজার তৈরি হওয়ার পর চেক করুন:**
```javascript
show users
```

---

## **Step 5: ইউজারের পাসওয়ার্ড পরিবর্তন করা**
```javascript
use admin
db.updateUser("myUser", { pwd: "newSecurePassword" });
```

---

## **Step 8: User মুছে ফেলা (Delete)**
```javascript
use admin
db.dropUser("myUser")
```

---

## **Step 9: Database Rename করা (MongoDB-তে সরাসরি Rename অপশন নেই)**
MongoDB-তে ডাটাবেস Rename করার কোনো সরাসরি কমান্ড নেই। তবে, **নতুন ডাটাবেস তৈরি করে পুরানো ডাটাবেস থেকে ডাটা কপি করে, পরে পুরানোটি ডিলিট করতে হয়।**

**Database Rename করার পদ্ধতি:**
```bash
mongodump --db oldDatabase --out /backup/
mongorestore --db newDatabase /backup/oldDatabase/
```

---

## **Step 10: Database Delete করা**
MongoDB-তে কোনো ডাটাবেস **ডিলিট করার আগে সেটিতে ঢুকতে হবে**:
```javascript
use myDatabase
db.dropDatabase()
```

ডাটাবেস ডিলিট হয়ে গেলে `show dbs` কমান্ড দিলে আর এটি দেখা যাবে না।

---

## **Step 11: MongoDB Restart, Stop ও Reload করা**
```bash
# MongoDB Restart
sudo systemctl restart mongod

# MongoDB Stop
sudo systemctl stop mongod

# MongoDB Start
sudo systemctl start mongod
```

---

## **সবকিছু ঠিকভাবে কাজ করছে কিনা চেক করা**
```bash
mongosh
```
তারপর লগইন করে চেক করুন:
```javascript
use myDatabase
show collections
db.myCollection.find()
```

---

## **Step 6: নির্দিষ্ট IP বা সকল IP থেকে Access দেওয়া (Firewall + Bind IP)**
MongoDB ডিফল্টভাবে শুধু লোকালহোস্ট (`127.0.0.1`) থেকে কানেক্ট হতে পারে। VPS-এর বাইরের কোনো সার্ভার থেকে অ্যাক্সেস দিতে হলে **bindIP পরিবর্তন ও firewall কনফিগার করতে হবে**।

### **1. MongoDB Config পরিবর্তন করা (Bind IP সেট করা)**
ফাইল ওপেন করুন:
```bash
sudo nano /etc/mongod.conf
```
নিচের অংশটি খুঁজুন:
```yaml
# network interfaces
net:
  bindIp: 127.0.0.1
```
**নির্দিষ্ট IP থেকে Access অনুমতি দিতে (যেমন: `192.168.1.100`)**
```yaml
net:
  bindIp: 127.0.0.1,192.168.1.100
```
**সকল IP থেকে Access দিতে (⚠️ সিকিউরিটির জন্য ঝুঁকিপূর্ণ)**
```yaml
net:
  bindIp: 0.0.0.0
```

পরিবর্তন সংরক্ষণ করতে:
👉 **CTRL + X → Y → ENTER**

### **2. Firewall Rule যোগ করা**
**নির্দিষ্ট IP অনুমতি দিতে (192.168.1.100)**
```bash
sudo ufw allow from 192.168.1.100 to any port 27017
```

**সকল IP কে Access দিতে (⚠️ নিরাপদ নয়)**
```bash
sudo ufw allow 27017/tcp
```

---

## **Step 7: Firewall Enable ও Status চেক করা**
```bash
sudo ufw enable
sudo ufw status
```
MongoDB Firewall rule চেক করতে:
```bash
sudo ufw status numbered
```


---

## **সংক্ষেপে গুরুত্বপূর্ণ কমান্ডগুলো** 
| কাজ | কমান্ড |
|------|--------------------------------|
| MongoDB Install | `sudo apt install -y mongodb-org` |
| MongoDB Start | `sudo systemctl start mongod` |
| MongoDB Status | `sudo systemctl status mongod` |
| MongoDB Shell Open | `mongosh` |
| নতুন Database তৈরি | `use myDatabase` |
| নতুন User তৈরি | `db.createUser({...})` |
| User Password Reset | `db.updateUser("user", {pwd: "newPass"})` |
| Firewall Enable | `sudo ufw enable` |
| নির্দিষ্ট IP Allow | `sudo ufw allow from 192.168.1.100 to any port 27017` |
| সকল IP Allow | `sudo ufw allow 27017/tcp` |
| Database Rename | `mongodump --db oldDB --out /backup/ && mongorestore --db newDB /backup/oldDB/` |
| Database Delete | `use myDatabase; db.dropDatabase()` |

---
এভাবে আপনি **MongoDB সেটআপ, ইউজার ম্যানেজমেন্ট, ডাটাবেস ম্যানেজমেন্ট, এবং Firewall কনফিগারেশন** করতে পারবেন! 🚀