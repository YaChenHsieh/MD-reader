### **User Pool Configuration: IHS Health System**  
I created a **Cognito User Pool** named **"IHS Health System"**. This user pool is configured according to the role-based access control system described in this document.  

- Users in this pool belong to specific **Cognito groups** based on their role in the system:
  - **IHS (SuperAdmin Group)** → Can manage all hospitals, ClinicAdmins, and Doctors.
  - **ClinicAdmin Group** → Can manage doctors within their hospital.
  - **Doctor Group** → Can upload and access their own medical documents.
  
- **Each user's hospital affiliation is stored using the `custom:hospital_id` attribute**, which allows ClinicAdmins to only manage doctors within their assigned hospital.

This user pool is integrated with **an Identity Pool** to allow role-based authentication and automatic IAM Role assignment when users log in.
---

Case:
Our **IHS group** has **two administrators (Angel and Jamie)** who are responsible for **creating ClinicAdmin accounts** for **Hospital A and Hospital B**.

- **Hospital A** has **two ClinicAdmins (a1, a2)** who manage **three doctors: AA1, AA2, and AA3**.
- **Hospital B** has **two ClinicAdmins (b1, b2)** who manage **two doctors: BB1 and BB2**.

This ensures that **each hospital's ClinicAdmins can only manage doctors within their assigned hospital**, while **IHS administrators have full control over all hospitals and their users**.

---

## **📌 Group and IAM Role Mapping**
| **Group Name**  | **Members** | **IAM Role** | **Permissions** |
|---------------|------------|--------------|-----------------------------|
| **IHS** (SuperAdmin) | Angel, Jamie | `IHSAdminIAMRole` | 1️⃣ **Manage all hospitals** <br> 2️⃣ **Create ClinicAdmin & Doctor accounts** <br> 3️⃣ **Access all hospital documents in S3 & DB** |
| **ClinicAdmin** (Hospital Admin) | a1, a2 (Hospital A), b1, b2 (Hospital B) | `ClinicAdminIAMRole` | 1️⃣ **Manage doctors within their hospital** <br> 2️⃣ **Create doctor accounts in Cognito (via "Create Doctor’s Doc" button in the frontend)** <br> 3️⃣ **Upload, read, and download doctor documents in S3 & DB** |
| **Doctor** (Physicians) | AA1, AA2, AA3 (Hospital A), BB1, BB2 (Hospital B) | `DoctorIAMRole` | 1️⃣ **Upload their own medical documents** <br> 2️⃣ **Read their uploaded documents in S3 & DB** |

---

## **📌 Cognito User Group Configuration**
| **Group Name** | **Requires `custom:hospital_id`?** | **Mapped IAM Role** |
|---------------|-----------------|------------------|
| **IHS** | ❌ No hospital_id needed | `IHSAdminIAMRole` |
| **ClinicAdmin** | ✅ **Must have `hospital_id`** | `ClinicAdminIAMRole` |
| **Doctor** | ✅ **Must have `hospital_id`** | `DoctorIAMRole` |

---

## **📌 User Group Assignments**
Each user should be assigned to the following Cognito groups:

| **User** | **Cognito Group** | **IAM Role** |
|----------|----------------|------------------|
| **Angel** | `IHS` | `IHSAdminIAMRole` |
| **Jamie** | `IHS` | `IHSAdminIAMRole` |
| **a1** | `ClinicAdmin` | `ClinicAdminIAMRole` |
| **a2** | `ClinicAdmin` | `ClinicAdminIAMRole` |
| **b1** | `ClinicAdmin` | `ClinicAdminIAMRole` |
| **b2** | `ClinicAdmin` | `ClinicAdminIAMRole` |
| **AA1** | `Doctor` | `DoctorIAMRole` |
| **AA2** | `Doctor` | `DoctorIAMRole` |
| **AA3** | `Doctor` | `DoctorIAMRole` |
| **BB1** | `Doctor` | `DoctorIAMRole` |
| **BB2** | `Doctor` | `DoctorIAMRole` |

---

## ** Identity Pool Role Mapping (IAM Role Assignment)**
Within **Cognito Identity Pool**, **Role Mapping** ensures that **Cognito groups automatically inherit the correct IAM Role**.

| **Cognito User Group** | **Identity Pool Role Mapping** | **IAM Role** |
|-------------------|--------------------------|----------|
| `IHS` | `cognito:groups = IHS` | `IHSAdminIAMRole` |
| `ClinicAdmin` | `cognito:groups = ClinicAdmin` | `ClinicAdminIAMRole` |
| `Doctor` | `cognito:groups = Doctor` | `DoctorIAMRole` |

✅ **This ensures that when users log in to Cognito, they automatically receive the correct IAM Role and the proper AWS access permissions (S3 & DynamoDB).**

---

## ** `custom:hospital_id` Applid time**
The `custom:hospital_id` attribute is mainly used for **ClinicAdmin & Doctor** to ensure:
1. **ClinicAdmin can only manage doctors within their own hospital.**
2. **Doctors can only access their own hospital’s records.**

### ** `custom:hospital_id` Assignment Workflow**
✅ **1. When IHS Creates a ClinicAdmin Account**
   - **Angel / Jamie (IHS Admins) create a ClinicAdmin in Cognito.**
   - In Cognito **manually assign `custom:hospital_id`**.
   - `a1, a2` have `custom:hospital_id = A`.
   - `b1, b2` have `custom:hospital_id = B`.

✅ **2. When ClinicAdmin Creates a Doctor Account**
   - **ClinicAdmin clicks "Create Doctor’s Doc" in the frontend** → backend API:
     - Creates a Doctor in **Cognito**.
     - **Automatically assigns ClinicAdmin’s `hospital_id`** to the new doctor.
   - Doctors will **inherit the corresponding `hospital_id`**:
     - **AA1, AA2, AA3** → `custom:hospital_id = A`
     - **BB1, BB2** → `custom:hospital_id = B`

✅ **3. When Doctor Logs in and Accesses S3 or DB**
   - Cognito **automatically attaches `hospital_id`**.
   - **Lambda / DynamoDB / S3 enforces access control based on `hospital_id`**.

 **To ensures that ClinicAdmins can only manage their own hospital's doctors and doctors cannot access data from other hospitals.**

---

## ** Missing Configurations**
✅ **Completed Configurations**
- [x] **User Pool** setup completed.
- [x] **User Groups** configured.
- [x] **Identity Pool** configured.
- [x] **IAM Role Mapping** completed.
- [x] **custom:hospital_id assignment mechanism designed.**

⚠ **Possible Missing Items**
1. **Lambda Function for `hospital_id` Verification**
   - When a ClinicAdmin **queries/manages** doctors, Lambda should:
     - Ensure **Doctor's `hospital_id` matches the ClinicAdmin's `hospital_id`**.
     - Prevent ClinicAdmin A from accessing ClinicAdmin B’s doctors.

2. **Frontend Login Flow Testing**
   - Verify **ClinicAdmin clicking "Create Doctor’s Doc" automatically passes `hospital_id`**.
   - Check **if Doctors can correctly read/upload their own files**.
   - Ensure **IHS can manage all hospitals' doctors**.

3. **API Authorization with `hospital_id`**
   - Ensure **all API calls validate `hospital_id`**.
   - **Prevent unauthorized ClinicAdmins or Doctors from accessing data from other hospitals.**

---

## **📌 Final Summary**
- ✅ **Cognito User Pool** handles **authentication**.
- ✅ **Cognito Identity Pool** manages **IAM permissions**.
- ✅ **Cognito Groups** control **user classification**.
- ✅ **IAM Role Mapping** ensures **Cognito Groups → IAM Role**.
- ✅ **custom:hospital_id** enforces **hospital-based access control**.
- ✅ **API and Lambda functions should validate `hospital_id`**.

### **System Security & Functionality Checklist**
✔ **IHS Admins** can manage **all hospitals**.  
✔ **ClinicAdmins** can **only manage doctors within their hospital**.  
✔ **Doctors** can **only access their own medical records**.  
✔ **Data between hospitals is fully isolated**.  
✔ **Cognito Groups & IAM automate permissions management**.

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

