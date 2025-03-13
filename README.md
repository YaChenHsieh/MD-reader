Case:
Our **IHS group** has **two administrators (Angel and Jamie)** who are responsible for **creating ClinicAdmin accounts** for **Hospital A and Hospital B**.

- **Hospital A** has **two ClinicAdmins (a1, a2)** who manage **three doctors: AA1, AA2, and AA3**.
- **Hospital B** has **two ClinicAdmins (b1, b2)** who manage **two doctors: BB1 and BB2**.

This ensures that **each hospital's ClinicAdmins can only manage doctors within their assigned hospital**, while **IHS administrators have full control over all hospitals and their users**.

---
### ** Latest System Design & Complete Workflow**
This document provides a **detailed system design** for managing authentication, access control, and data permissions in the **IHS Health System** using **Cognito User Pool, Identity Pool, IAM, DynamoDB, and APIs**. It ensures that **ClinicAdmin, Doctor, and IHS (SuperAdmin)** roles function correctly while addressing **doctor hospital transfers (`Changing Hospital`)** and account lifecycle management (`Create / Archive ClinicAdmin and Doctor Accounts`).

---

## ** 1. Cognito Configuration**
### ** Cognito User Pool**
Name: **IHS Health System**  

| **Cognito Group** | **Purpose** | **Description** |
|----------------|--------|-----------------------------|
| **IHS (SuperAdmin)** | **Manage all hospitals** | Can create **ClinicAdmin and Doctor accounts** and manage all data |
| **ClinicAdmin** | **Manage doctors within a hospital** | Can only manage **doctors in their assigned hospital** |
| **Doctor** | **Doctor accounts** | **Can only manage their own medical documents** (🚨 **Doctors cannot access documents of other doctors, even in the same hospital!**). |

---

### ** Cognito User Group & IAM Role Mapping**
| **Cognito Group** | **IAM Role (Role Mapping)** | **Permissions** |
|---------------|------------------|-----------------------------|
| **IHS (SuperAdmin)** | `IHSAdminIAMRole` | 🚀 **Manage all hospitals, ClinicAdmin, and Doctor accounts** |
| **ClinicAdmin** | `ClinicAdminIAMRole` | 🏥 **Manage doctors within their hospital** |
| **Doctor** | `DoctorIAMRole` | 🩺 **Only access their own medical records** |

---

### ** Cognito User Attributes**
| **Attribute Name** | **Type** | **Description** |
|-----------|------|--------------------------|
| `email` | String | Doctor / Admin Email |
| `custom:hospital_id` | String | **Only set for ClinicAdmin (🚨 Not used for Doctors)** |
| `sub` | String | **Cognito User ID (Automatically available in the ID Token, no need to store as an attribute)** |

**`custom:hospital_id` is only assigned to ClinicAdmins. Doctors' `hospital_id` is stored in the `Provider Table`.**

---

## ** 2. IAM Role & Permission Management**
### ** IAM Roles & Permissions**
| **IAM Role** | **Associated Cognito Group** | **Permissions** |
|------------|----------------|-----------------------------|
| `IHSAdminIAMRole` | IHS | 🚀 **Manage all hospitals, ClinicAdmin, and Doctor accounts** |
| `ClinicAdminIAMRole` | ClinicAdmin | 🏥 **Manage doctors within their hospital** |
| `DoctorIAMRole` | Doctor | 🩺 **Only access their own medical documents** |

---

### ** IAM Permission Scope**
#### **IHSAdminIAMRole (SuperAdmin)**
- **Manage all Cognito users (including ClinicAdmin and Doctors)**
- **Access all hospital documents in DynamoDB & S3**
- **Create / Archive ClinicAdmin and Doctor accounts**
- **Update `hospital_id` (doctor hospital transfers)**

#### **ClinicAdminIAMRole (Hospital Admin)**
- **Access doctors in their hospital (`hospital_id` filter in DynamoDB)**
- **Create / Query / Update doctors within their hospital**
- **Upload / Read / Download hospital-related documents**

#### **DoctorIAMRole (Doctors)**
**Doctors can only access their own records. They cannot view files from other doctors, even within the same hospital.**
- **Access only their `providerId` data**
- **Upload / Read / Download their own medical records**

---

## ** 3. DynamoDB Table Design**
### ** `Provider Table` (Doctor Data)**
| **Column** | **Type** | **Description** |
|---------|------|----------------------|
| `providerId` | String (PK) | **Cognito User ID** |
| `email` | String | Doctor’s Email |
| `firstName` | String | First Name |
| `lastName` | String | Last Name |
| `dateOfBirth` | String | Date of Birth |
| `gender` | String | Gender |
| `address` | String | Address |
| `city` | String | City |
| `state` | String | State |
| `zipCode` | String | ZIP Code |
| `hospital_id` | String | **Current hospital ID where the doctor is employed** |

**🚀 `hospital_id` is stored directly in the `Provider Table`, removing the need for a separate mapping table.**

---

## ** 4. API Design**
| **API Path** | **Method** | **Purpose** | **Access Level** |
|-------------|------|---------|-------------------|
| `/createDoctor` | `POST` | **Create a new doctor account** | IHS & ClinicAdmin |
| `/getDoctor` | `GET` | **Retrieve doctor info** | IHS & ClinicAdmin |
| `/updateDoctorHospital` | `PUT` | **Update `hospital_id` (doctor transfer)** | IHS & ClinicAdmin |
| `/createCase` | `POST` | **Create a new case** | ClinicAdmin |
| `/getDoctorCases` | `GET` | **Retrieve doctor’s cases** | ClinicAdmin & Doctor |

**All APIs check `hospital_id` to enforce access control.**

---

## ** 5. Complete System Workflow**
### **✅ 1. `Create Doctor’s Doc`**
**ClinicAdmin enters the doctor’s email**  
**System queries `Provider Table` to check if the doctor exists**
   - ✅ **Found** → **Update `hospital_id`** to assign the doctor to the new hospital.
   - ❌ **Not found** → **Frontend displays "User not found" with a `Create User` button.**
**ClinicAdmin clicks `Create User`**
   - **Create a new Cognito account**
   - **Assign `providerId = Cognito User ID`**
   - **Add the doctor’s record to `Provider Table`**
   - **Set `hospital_id`**
**Account is created; ClinicAdmin can now create cases.**

---

### **✅ 2. `Create Case`**
**ClinicAdmin creates a case for a doctor**  
**System automatically associates the case with the doctor’s `hospital_id`**  
**Ensures that only the assigned doctor can access the case.**

---

### **✅ 3. Doctor Hospital Transfer**
**ClinicAdmin requests to transfer a doctor**
**System updates `hospital_id`**
   - **Revokes access from the previous hospital**
   - **Grants access to the new hospital**
**Doctor logs in and can only access their own records.**

---

### **✅ 4. Doctor Login & Access**
**Doctor logs into Cognito**
**System reads `hospital_id` from `Provider Table`**
**Doctor can only access their own `providerId` medical records (🚨 No access to other doctors' records, even within the same hospital).**

---

### Questions for the Specialist

1. **Is my current system configuration correct based on our system's usage rules?**  
   - I have set up **one Cognito user pool** with **three groups** (IHS, ClinicAdmin, and Doctor), each with different permissions.  
   - I am using `hospital_id` to differentiate and restrict access between different hospitals.  
   - Is this approach feasible? Are there any missing considerations or details I should be aware of?  
   - In the future, if the number of hospitals exceeds 100 and each hospital has more than 100 doctors, will there be **management scalability issues**?  

2. **Is there a better design than my current setup?**  
   - From both a **management** and **cost-efficiency** perspective, is there a more optimal way to structure this?  
   - I understand that by using `hospital_id` for access control, I will need to **validate it via Lambda functions**, which may result in **API call costs**.  
   - Do you have any recommendations on how to **manage costs more effectively** while maintaining a similar level of security and access control?  

3. **Can permissions be customized at the individual user level within each Cognito group?**  
   - For example, in the **IHS group**, there are two employees, but one of them is a **supervisor**.  
   - Can we configure the supervisor to have **higher-level permissions** than other users in the same group?  
   - If so, what is the best way to achieve this? Can this be done by assigning **custom IAM permissions to a specific user**?  

4. **What mechanisms can we use to prevent privilege abuse?**  
   - Besides **logging** user actions such as document uploads, downloads, and feature usage,  
   - Is there a way to **set up alerts** for administrators (IHS or ClinicAdmin supervisors) when suspicious activity occurs?  
   - What options are available in **AWS Cognito or IAM** to notify managers of **unusual behaviors** or **policy violations** in real time?  

