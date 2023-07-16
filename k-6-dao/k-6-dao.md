# ComicCrafter Team K-6-DAO SMART CONTRACT SECURITY REVIEW
## *Overview*
Good job in implementing this complex project. We understand that your contracts are not fully completed because we found some incomplete logic. We tried to give you a complete feedback to help you improve it but due to lack of documentation and the fact that the code is not yet finished, we might misunderstand some part or get confused and this in turn will affect our report. 
As a general comment, due to the unclear architecture  there are some repeated logic e.g  getting students' profiles and tutors' profiles in `accessStudentProfile`, `getProfile` and `accessTutorProfile` in `MainContract` and `Administration` calling these function from `MainContract` is actually is calling them in `Administration` which circling back to  `MainContract` to call `getStudentProfile` and `getTutorProfile`. This is just a gas waisting. `getProfile`  and `editDAOCode` in `MainContract` and `Administration` also has similar issue. 
It's very important to have a access control in your contracts to prevent unauthorized access to your contracts. We found some functions that are not protected by access control e.g `setMainContract` in `Administration`,  `setAdministrationContract` in `MainContract`  and `editDAOCode` in `MainContract` and `Administration` .
We also found uncompleted logic e.g .  `editDAOCode` in `MainContract` and `Administration` 

During reviewing the code we found some vulnerabilities in different severity levels explained below 


## Project Summary

A decentralized autonomous organization (DAO) focused on K-6 education. The DAO facilitates interactions between students, tutors, and administrators to enhance decentralized learning for elementary school students.


## Report Scope
- The scope of the audit was to verify the security of the smart contracts and the codebase of the K-6 DAO project. The audit was conducted on the following contracts:
  - Student Contract
  - Tutor Contract
  - Administrator Contract
  - Main Contract

## List of Findings
- 3 high severity issues.
- 3 medium severity issues.
- 1 Technical issue.
  

| # | Title | Label | Severity |
| --- | --- | --- | --- |
| [H-1] | UnAuthorized access to `setMainContract` in `Administration`,  `setAdministrationContract` in `MainContract`  and `editDAOCode` in `MainContract` and `Administration` | `access control` | `High` |
| [H-2] | Only one student and one tutor could be registered via `MainContract`  causing a DOS. | `DoS` | `High` |
| [H-3] | Confidential data shouldn't be stored in the blockchain. | `Confidentiality` | `High` |
| [M-1] | Student can add himself as a tutor and vice versa. | `Access Control` | `Medium` |
| [M-2] | Student can assign himself as a tutor and vice versa.* | `Access Control` | `Medium` |
| [M-3] | `SelectSubject` function is not storing any data related to the subject. |  | `Medium` |
| [Technical-Error-1] | `getStudentProfile` and  `getTutorProfile` in `MainContract` are calling undefined functions. | `Technical issue` | `Medium` |
 |   |   |   |   | 

## Findings
 ### [H-1] UnAuthorized access to `setMainContract` in `Administration`,  `setAdministrationContract` in `MainContract`  and `editDAOCode` in `MainContract` and `Administration` 

Malicious actor can set `MainContract` address to any  contract address if the attacker could watch the contract deployment and call `setMainContract`. The attacker also might front run it . Thus, leaving this function unprotected leads to unexpected behavior. same applies to `setAdministrationContract` in `MainContract` and `editDAOCode` in `MainContract` and `Administration` . 
```solidity
/// "Administration"
   function setMainContract(address _mainContract) external {
        require(mainContract == address(0), "Main contract is already set."); // Check if the main contract is not already set
        mainContract = _mainContract; // Set the address of the main contract
        emit MainContractSet(mainContract); // Emit the MainContractSet event
    }
      function editDAOCode(uint256 _parameters) external {
        require(mainContract != address(0), "Main contract is not set."); // Check if the main contract is set
        MainContract(mainContract).editDAOCode(_parameters); // Call the editDAOCode function of the main contract
    }

    /// "MainContract"

    function setAdministrationContract(address _administrationContract) external {
        administrationContract = _administrationContract; // Set the address of the administration contract
    }
       function editDAOCode(uint256 _parameters) external {
        require(administrationContract != address(0), "Administration contract is not set."); // Check if the administration contract is set
        Administration(administrationContract).editDAOCode(_parameters); // Call the editDAOCode function of the administration contract
    }
```

### [H-2] Only one student and one tutor could be registered via `MainContract`  causing a DOS.
Based on what's mentioned in `readme` , The `MainContract` acts as the central contract that connects the student, tutor, and administrator contracts. By Calling `registerStudent` and   `registerTutor` you are adding the `MainContract` address as tutor and as student because you are checking the `msg.sender` to make sure it's not registered before and assign it as student/tutor 
```solidity
/// "StudentContract"
   function register(string memory _email, string memory _password) external {
     
        require(!students[msg.sender].isRegistered, "Student is already registered.");

        // Perform additional registration checks if needed

        students[msg.sender].isRegistered = true;

        // Optionally, store additional details about the student

        emit StudentRegistered(msg.sender);
    }

    ///  in  "TutorContract"
       function register() external {
        require(!tutors[msg.sender].isRegistered, "Tutor is already registered.");

        // Perform additional registration checks if needed

        tutors[msg.sender].isRegistered = true;

        // Optionally, store additional details about the tutor securely

        emit TutorRegistered(msg.sender);
    }

```
### [H-3]  Confidential data shouldn't be stored in the blockchain.
Blockchain is a public ledger and all the data stored in the blockchain is visible to everyone. So, storing confidential data in the blockchain is not a good practice. In the `StudentContract` confidential data like `password` and `email` are stored in the blockchain which is not a good practice. These parameters is not used in contract  but we assume you are going to implement them since the contracts are stull under development . If you are not going to use tem, we highly encourage you to remove them as they are a waste of gas (unused variables/parameters). 
 

### *[M-1] Student can add himself as a tutor and vice versa.*
Student can add himself as a tutor by calling the `register` function in the `Tutor` contract. This is because the `register` function does not check if the caller is a student or not. The same applies to the tutor. Tutor can add himself as a student by calling the `register` function in the `Student` contract. This is because the `register` function does not check if the caller is a tutor or not.


### *[M-2] Student can assign himself as a tutor and vice versa.*
Student can assign himself as a tutor by calling the `selectSubject` function in the `StudentContract` and pass his address in  function parameter. The same applies to the tutor.



### *[M-3] `SelectSubject` function is not storing any data related to the subject.*
The `selectSubject` function in `TutorContract` and `StudentContract` are not storing any data related to the subject. It is just marking the subject as selected. No way to know which subject is selected by the student and matching the subject with the tutor.Also, the logic in the `selectSubject` function is unclear. We couldn't understand what the function is trying to do and why tutor should assign the student address. Making the function unclear and not storing any data related to the subject makes the DAO fails to achieve its goal. 

### [Technical-Error-1]  `getStudentProfile` and  `getTutorProfile` in `MainContract` are calling undefined functions
`getStudentProfile` and  `getTutorProfile` in `MainContract`  and `getProfile` in `Administration`are calling undefined functions. The functions are not defined in the `StudentContract` and `TutorContract` respectively. This is causing a compilation error.
 

 
 Audit methdology used in our assessment:
- Manual Review
  
## Code Commit

- [https://github.com/0xBcamp/K-6-DAO_June2023/commit/042efda9eaa947a9276ceb92f2c1505181c32e4a]

Appreciate your time and effort in reading the report. We are looking forward to hearing from you soon.



