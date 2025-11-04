Part (a): Dependency Injection in Spring Using Java-Based Configuration
//Course.java
package com.example.model;

public class Course {
    private String courseName;

    public Course(String courseName) {
        this.courseName = courseName;
    }

    public String getCourseName() {
        return courseName;
    }

    public void displayCourse() {
        System.out.println("Course Name: " + courseName);
    }
}
//Student.java

package com.example.model;

public class Student {
    private Course course;

    // Dependency injected through constructor
    public Student(Course course) {
        this.course = course;
    }

    public void displayInfo() {
        System.out.println("Student enrolled in:");
        course.displayCourse();
    }
}

//AppConfig.java

package com.example.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import com.example.model.*;

@Configuration
public class AppConfig {

    @Bean
    public Course courseBean() {
        return new Course("Spring Framework with Hibernate");
    }

    @Bean
    public Student studentBean() {
        // Inject Course dependency
        return new Student(courseBean());
    }
}

//MainApp.java
package com.example;

import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import com.example.config.AppConfig;
import com.example.model.Student;

public class MainApp {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context =
                new AnnotationConfigApplicationContext(AppConfig.class);

        Student student = context.getBean(Student.class);
        student.displayInfo();

        context.close();
    }
}


//Part (b): Hibernate Application for Student CRUD Operations
//hibernate.cfg.xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE hibernate-configuration PUBLIC 
 "-//Hibernate/Hibernate Configuration DTD 3.0//EN"
 "http://hibernate.sourceforge.net/hibernate-configuration-3.0.dtd">
<hibernate-configuration>
    <session-factory>
        <property name="hibernate.connection.driver_class">com.mysql.cj.jdbc.Driver</property>
        <property name="hibernate.connection.url">jdbc:mysql://localhost:3306/testdb</property>
        <property name="hibernate.connection.username">root</property>
        <property name="hibernate.connection.password">your_password</property>
        <property name="hibernate.dialect">org.hibernate.dialect.MySQL8Dialect</property>
        <property name="hibernate.hbm2ddl.auto">update</property>
        <mapping class="com.example.hibernate.Student"/>
    </session-factory>
</hibernate-configuration>
//Student.java

package com.example.hibernate;

import jakarta.persistence.*;

@Entity
@Table(name = "student")
public class Student {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private int id;

    @Column(length = 50)
    private String name;

    private String department;
    private double marks;

    public Student() {}

    public Student(String name, String department, double marks) {
        this.name = name;
        this.department = department;
        this.marks = marks;
    }

    // Getters and Setters
    public int getId() { return id; }
    public String getName() { return name; }
    public String getDepartment() { return department; }
    public double getMarks() { return marks; }

    public void setName(String name) { this.name = name; }
    public void setDepartment(String department) { this.department = department; }
    public void setMarks(double marks) { this.marks = marks; }

    @Override
    public String toString() {
        return id + " - " + name + " (" + department + ") : " + marks;
    }
}

//HibernateUtil.java
package com.example.hibernate;

import org.hibernate.SessionFactory;
import org.hibernate.cfg.Configuration;

public class HibernateUtil {
    private static final SessionFactory sessionFactory =
        new Configuration().configure().buildSessionFactory();

    public static SessionFactory getSessionFactory() {
        return sessionFactory;
    }
}

//MainApp.java
package com.example.hibernate;

import org.hibernate.Session;
import org.hibernate.Transaction;

public class MainApp {
    public static void main(String[] args) {
        Session session = HibernateUtil.getSessionFactory().openSession();
        Transaction tx = null;

        try {
            tx = session.beginTransaction();

            // CREATE
            Student s1 = new Student("Alice", "IT", 89.5);
            session.save(s1);

            // READ
            Student fetched = session.get(Student.class, s1.getId());
            System.out.println("Fetched: " + fetched);

            // UPDATE
            fetched.setMarks(92.0);
            session.update(fetched);

            // DELETE
            session.delete(fetched);

            tx.commit();
            System.out.println("CRUD operations completed successfully.");
        } catch (Exception e) {
            if (tx != null) tx.rollback();
            e.printStackTrace();
        } finally {
            session.close();
        }
    }
}


//Part (c): Transaction Management Using Spring and Hibernate
//Account.java
package com.example.model;

import jakarta.persistence.*;

@Entity
@Table(name = "account")
public class Account {
    @Id
    private int accNo;

    private String name;
    private double balance;

    public Account() {}
    public Account(int accNo, String name, double balance) {
        this.accNo = accNo;
        this.name = name;
        this.balance = balance;
    }

    public int getAccNo() { return accNo; }
    public String getName() { return name; }
    public double getBalance() { return balance; }

    public void setBalance(double balance) { this.balance = balance; }
}

//AccountDAO.java

package com.example.dao;

import org.hibernate.SessionFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Repository;
import com.example.model.Account;

@Repository
public class AccountDAO {

    @Autowired
    private SessionFactory sessionFactory;

    public Account getAccount(int accNo) {
        return sessionFactory.getCurrentSession().get(Account.class, accNo);
    }

    public void updateAccount(Account account) {
        sessionFactory.getCurrentSession().update(account);
    }
}



//MainApp.java
//BankService.java
package com.example.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import com.example.dao.AccountDAO;
import com.example.model.Account;

@Service
public class BankService {

    @Autowired
    private AccountDAO accountDAO;

    @Transactional
    public void transferMoney(int fromAcc, int toAcc, double amount) {
        Account sender = accountDAO.getAccount(fromAcc);
        Account receiver = accountDAO.getAccount(toAcc);

        if (sender.getBalance() < amount) {
            throw new RuntimeException("Insufficient funds!");
        }

        sender.setBalance(sender.getBalance() - amount);
        receiver.setBalance(receiver.getBalance() + amount);

        accountDAO.updateAccount(sender);
        accountDAO.updateAccount(receiver);

        System.out.println("Transaction successful!");
    }
}

//AppConfig.java
package com.example.config;

import org.springframework.context.annotation.*;
import org.springframework.orm.hibernate5.*;
import org.springframework.transaction.annotation.EnableTransactionManagement;
import org.hibernate.SessionFactory;
import javax.sql.DataSource;
import org.springframework.jdbc.datasource.DriverManagerDataSource;
import com.example.model.Account;

@Configuration
@ComponentScan(basePackages = "com.example")
@EnableTransactionManagement
public class AppConfig {

    @Bean
    public DataSource dataSource() {
        DriverManagerDataSource ds = new DriverManagerDataSource();
        ds.setDriverClassName("com.mysql.cj.jdbc.Driver");
        ds.setUrl("jdbc:mysql://localhost:3306/testdb");
        ds.setUsername("root");
        ds.setPassword("your_password");
        return ds;
    }

    @Bean
    public LocalSessionFactoryBean sessionFactory() {
        LocalSessionFactoryBean factory = new LocalSessionFactoryBean();
        factory.setDataSource(dataSource());
        factory.setPackagesToScan("com.example.model");
        factory.getHibernateProperties().put("hibernate.dialect", "org.hibernate.dialect.MySQL8Dialect");
        factory.getHibernateProperties().put("hibernate.hbm2ddl.auto", "update");
        return factory;
    }

    @Bean
    public HibernateTransactionManager transactionManager(SessionFactory sf) {
        return new HibernateTransactionManager(sf);
    }
}
//MainApp.java
package com.example;

import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import com.example.config.AppConfig;
import com.example.service.BankService;

public class MainApp {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context =
                new AnnotationConfigApplicationContext(AppConfig.class);

        BankService bankService = context.getBean(BankService.class);

        // Example transaction
        try {
            bankService.transferMoney(101, 102, 1000.0);
        } catch (Exception e) {
            System.out.println("Transaction failed: " + e.getMessage());
        }

        context.close();
    }
}
