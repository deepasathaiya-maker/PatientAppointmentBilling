/*
UML (brief, textual):

Person (abstract)
|- patientId: String
|- name: String
|- phone: String
|- email: String
+ getters/setters

Patient extends Person
|- dateOfBirth: String

Doctor extends Person
|- specialization: String
|- fee: double

Appointment
|- id: String
|- patient: Patient
|- doctor: Doctor
|- slot: java.time.LocalDateTime
|- status: AppointmentStatus {SCHEDULED, COMPLETED, CANCELLED}

Consultation
|- id: String
|- appointment: Appointment
|- notes: String
|- prescriptions: List<PrescriptionItem>

PrescriptionItem
|- name: String
|- quantity: int
|- unitPrice: double

Invoice
|- id: String
|- consultation: Consultation
|- consultationFee: double
|- itemsTotal: double
|- taxRate: double
|- total: double
|- paid: boolean

Payment
|- id: String
|- invoice: Invoice
|- amount: double
|- timestamp: LocalDateTime

Main: ClinicApp (menu-driven console)

Business rules implemented in code comments.

Run instructions (also at bottom of file):
1. Create a Java project in Eclipse (use package com.csdept.clinic or default).
2. Create a new class file named Assignment12_PatientAppointmentBilling.java and paste the entire contents of this file.
3. Run the class as Java Application.

*/

package com.csdept.clinic;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.*;
import java.util.concurrent.atomic.AtomicInteger;

// --- Domain classes ---

abstract class Person {
    protected String id;
    protected String name;
    protected String phone;
    protected String email;

    public Person(String id, String name, String phone, String email) {
        this.id = id;
        this.name = name;
        this.phone = phone;
        this.email = email;
    }

    public String getId() { return id; }
    public String getName() { return name; }
    public String getPhone() { return phone; }
    public String getEmail() { return email; }

    @Override
    public String toString() {
        return String.format("%s (id=%s, phone=%s, email=%s)", name, id, phone, email);
    }
}

class Patient extends Person {
    private String dob;

    public Patient(String id, String name, String phone, String email, String dob) {
        super(id,name,phone,email);
        this.dob = dob;
    }

    public String getDob() { return dob; }
}

class Doctor extends Person {
    private String specialization;
    private double fee;

    public Doctor(String id, String name, String phone, String email, String specialization, double fee) {
        super(id,name,phone,email);
        this.specialization = specialization;
        this.fee = fee;
    }

    public String getSpecialization() { return specialization; }
    public double getFee() { return fee; }

    @Override
    public String toString() {
        return String.format("Dr. %s (id=%s, %s, fee=%.2f)", name, id, specialization, fee);
    }
}

enum AppointmentStatus { SCHEDULED, COMPLETED, CANCELLED }

class Appointment {
    private String id;
    private Patient patient;
    private Doctor doctor;
    private LocalDateTime slot;
    private AppointmentStatus status;

    public Appointment(String id, Patient patient, Doctor doctor, LocalDateTime slot) {
        this.id = id;
        this.patient = patient;
        this.doctor = doctor;
        this.slot = slot;
        this.status = AppointmentStatus.SCHEDULED;
    }

    public String getId() { return id; }
    public Patient getPatient() { return patient; }
    public Doctor getDoctor() { return doctor; }
    public LocalDateTime getSlot() { return slot; }
    public AppointmentStatus getStatus() { return status; }
    public void setStatus(AppointmentStatus status) { this.status = status; }

    @Override
    public String toString() {
        return String.format("Appointment %s | %s with %s at %s | %s",
            id, patient.getName(), doctor.getName(), slot.format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm")), status);
    }
}

class PrescriptionItem {
    private String name;
    private int quantity;
    private double unitPrice;

    public PrescriptionItem(String name, int quantity, double unitPrice) {
        this.name = name;
        this.quantity = quantity;
        this.unitPrice = unitPrice;
    }

    public double total() { return quantity * unitPrice; }
    public String getName() { return name; }
    public int getQuantity() { return quantity; }
    public double getUnitPrice() { return unitPrice; }

    @Override
    public String toString() {
        return String.format("%s x%d @ %.2f = %.2f", name, quantity, unitPrice, total());
    }
}

class Consultation {
    private String id;
    private Appointment appointment;
    private String notes;
    private List<PrescriptionItem> prescriptions = new ArrayList<>();

    public Consultation(String id, Appointment appointment, String notes) {
        this.id = id;
        this.appointment = appointment;
        this.notes = notes;
    }

    public String getId() { return id; }
    public Appointment getAppointment() { return appointment; }
    public String getNotes() { return notes; }
    public List<PrescriptionItem> getPrescriptions() { return prescriptions; }

    public void addPrescription(PrescriptionItem item) { prescriptions.add(item); }

    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder();
        sb.append(String.format("Consultation %s for %s with %s\n", id, appointment.getPatient().getName(), appointment.getDoctor().getName()));
        sb.append("Notes: ").append(notes).append("\n");
        if (prescriptions.isEmpty()) sb.append("No prescriptions.\n");
        else {
            sb.append("Prescriptions:\n");
            for (PrescriptionItem it: prescriptions) sb.append("  - ").append(it.toString()).append('\n');
        }
        return sb.toString();
    }
}

class Invoice {
    private String id;
    private Consultation consultation;
    private double consultationFee;
    private double itemsTotal;
    private double taxRate; // 0.0-1.0
    private boolean closed = false;

    public Invoice(String id, Consultation consultation, double taxRate) {
        this.id = id;
        this.consultation = consultation;
        this.consultationFee = consultation.getAppointment().getDoctor().getFee();
        this.taxRate = taxRate;
        computeItemsTotal();
    }

    private void computeItemsTotal() {
        this.itemsTotal = consultation.getPrescriptions().stream().mapToDouble(PrescriptionItem::total).sum();
    }

    public double getSubtotal() { return consultationFee + itemsTotal; }
    public double getTax() { return getSubtotal() * taxRate; }
    public double getTotal() { return getSubtotal() + getTax(); }

    public String getId() { return id; }
    public Consultation getConsultation() { return consultation; }

    public boolean isClosed() { return closed; }
    public void markClosed() { this.closed = true; }

    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder();
        sb.append(String.format("Invoice %s\n", id));
        sb.append(String.format("Consultation fee: %.2f\n", consultationFee));
        sb.append(String.format("Items total: %.2f\n", itemsTotal));
        sb.append(String.format("Tax (%.2f%%): %.2f\n", taxRate*100, getTax()));
        sb.append(String.format("TOTAL: %.2f\n", getTotal()));
        sb.append("Status: ").append(closed?"PAID":"UNPAID");
        return sb.toString();
    }
}

class Payment {
    private String id;
    private Invoice invoice;
    private double amount;
    private LocalDateTime timestamp;

    public Payment(String id, Invoice invoice, double amount) {
        this.id = id;
        this.invoice = invoice;
        this.amount = amount;
        this.timestamp = LocalDateTime.now();
    }

    public String getId() { return id; }
    public Invoice getInvoice() { return invoice; }
    public double getAmount() { return amount; }

    @Override
    public String toString() {
        return String.format("Payment %s | Invoice=%s | Amount=%.2f | At=%s", id, invoice.getId(), amount, timestamp.format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm")));
    }
}

// --- Simple in-memory repository and utilities ---

class IdGenerator {
    private static AtomicInteger counter = new AtomicInteger(100);
    public static String next(String prefix) { return prefix + counter.getAndIncrement(); }
}

public class Assignment12_PatientAppointmentBilling {
    private static Scanner sc = new Scanner(System.in);

    // repositories
    private static Map<String, Patient> patients = new LinkedHashMap<>();
    private static Map<String, Doctor> doctors = new LinkedHashMap<>();
    private static Map<String, Appointment> appointments = new LinkedHashMap<>();
    private static Map<String, Consultation> consultations = new LinkedHashMap<>();
    private static Map<String, Invoice> invoices = new LinkedHashMap<>();
    private static Map<String, Payment> payments = new LinkedHashMap<>();

    private static final double TAX_RATE = 0.12; // 12% tax

    public static void main(String[] args) {
        seedSampleData();
        menuLoop();
    }

    private static void seedSampleData() {
        Patient p1 = new Patient("P001","Deepa","+91-99999-00001","deepa@example.com","1999-01-01");
        Patient p2 = new Patient("P002","Arun","+91-99999-00002","arun@example.com","1998-02-12");
        patients.put(p1.getId(), p1);
        patients.put(p2.getId(), p2);

        Doctor d1 = new Doctor("D001","Ravi","+91-88888-0001","ravi@clinic.com","General Physician",500.0);
        Doctor d2 = new Doctor("D002","Meera","+91-88888-0002","meera@clinic.com","Dermatologist",800.0);
        doctors.put(d1.getId(), d1);
        doctors.put(d2.getId(), d2);

        // sample appointment
        LocalDateTime slot = LocalDateTime.now().plusDays(1).withHour(10).withMinute(0).withSecond(0).withNano(0);
        Appointment a = new Appointment("A100", p1, d1, slot);
        appointments.put(a.getId(), a);
    }

    // Main menu
    private static void menuLoop() {
        while (true) {
            System.out.println("\n--- Patient Appointment & Billing ---");
            System.out.println("1. Add Patient");
            System.out.println("2. Add Doctor");
            System.out.println("3. Schedule Appointment");
            System.out.println("4. Record Consultation (for completed appointment)");
            System.out.println("5. Generate Invoice");
            System.out.println("6. Record Payment");
            System.out.println("7. List Appointments");
            System.out.println("8. Outstanding Dues Report");
            System.out.println("9. Exit");
            System.out.print("Choose: ");
            String choice = sc.nextLine().trim();
            try {
                switch (choice) {
                    case "1": addPatient(); break;
                    case "2": addDoctor(); break;
                    case "3": scheduleAppointment(); break;
                    case "4": recordConsultation(); break;
                    case "5": generateInvoice(); break;
                    case "6": recordPayment(); break;
                    case "7": listAppointments(); break;
                    case "8": outstandingDuesReport(); break;
                    case "9": System.out.println("Goodbye"); return;
                    default: System.out.println("Invalid choice");
                }
            } catch (Exception ex) {
                System.out.println("Error: " + ex.getMessage());
            }
        }
    }

    private static void addPatient() {
        System.out.println("-- Add Patient --");
        System.out.print("Name: "); String name = sc.nextLine();
        System.out.print("Phone: "); String phone = sc.nextLine();
        System.out.print("Email: "); String email = sc.nextLine();
        System.out.print("DOB (yyyy-mm-dd): "); String dob = sc.nextLine();
        String id = IdGenerator.next("P");
        Patient p = new Patient(id, name, phone, email, dob);
        patients.put(id, p);
        System.out.println("Patient added: " + p);
    }

    private static void addDoctor() {
        System.out.println("-- Add Doctor --");
        System.out.print("Name: "); String name = sc.nextLine();
        System.out.print("Phone: "); String phone = sc.nextLine();
        System.out.print("Email: "); String email = sc.nextLine();
        System.out.print("Specialization: "); String spec = sc.nextLine();
        System.out.print("Fee: "); double fee = Double.parseDouble(sc.nextLine());
        String id = IdGenerator.next("D");
        Doctor d = new Doctor(id, name, phone, email, spec, fee);
        doctors.put(id, d);
        System.out.println("Doctor added: " + d);
    }

    private static void scheduleAppointment() {
        System.out.println("-- Schedule Appointment --");
        System.out.println("Available Patients:");
        patients.values().forEach(p->System.out.println(p.getId()+" - "+p.getName()));
        System.out.print("Enter patient id: "); String pid = sc.nextLine().trim();
        Patient p = patients.get(pid);
        if (p==null) { System.out.println("Patient not found"); return; }

        System.out.println("Available Doctors:");
        doctors.values().forEach(d->System.out.println(d.getId()+" - "+d.getName()+" ("+d.getSpecialization()+")"));
        System.out.print("Enter doctor id: "); String did = sc.nextLine().trim();
        Doctor d = doctors.get(did);
        if (d==null) { System.out.println("Doctor not found"); return; }

        System.out.print("Enter slot (yyyy-MM-dd HH:mm): ");
        String slotStr = sc.nextLine().trim();
        LocalDateTime slot;
        try { slot = LocalDateTime.parse(slotStr, DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm")); }
        catch (Exception ex) { System.out.println("Invalid date format"); return; }

        // Business rule: Appointments can be scheduled only in available slots.
        // Here we check for conflicts for the same doctor at the exact same time.
        boolean conflict = appointments.values().stream()
            .anyMatch(a -> a.getDoctor().getId().equals(did) && a.getSlot().equals(slot) && a.getStatus()==AppointmentStatus.SCHEDULED);
        if (conflict) {
            System.out.println("Selected slot is not available for this doctor.");
            return;
        }

        String aid = IdGenerator.next("A");
        Appointment appt = new Appointment(aid, p, d, slot);
        appointments.put(aid, appt);
        System.out.println("Appointment scheduled: " + appt);
    }

    private static void recordConsultation() {
        System.out.println("-- Record Consultation --");
        System.out.println("Scheduled Appointments:");
        appointments.values().stream().filter(a->a.getStatus()==AppointmentStatus.SCHEDULED).forEach(a->System.out.println(a.getId()+" - " + a));
        System.out.print("Enter appointment id to mark as completed and record consultation: ");
        String aid = sc.nextLine().trim();
        Appointment a = appointments.get(aid);
        if (a==null) { System.out.println("Appointment not found"); return; }

        // Business rule: consultation can be recorded only for completed appointments.
        // We'll mark the appointment completed now, and only then create the consultation.
        a.setStatus(AppointmentStatus.COMPLETED);
        System.out.print("Enter consultation notes: "); String notes = sc.nextLine();
        String cid = IdGenerator.next("C");
        Consultation cons = new Consultation(cid, a, notes);

        // Allow adding prescriptions
        while (true) {
            System.out.print("Add prescription item? (y/n): "); String yn = sc.nextLine().trim().toLowerCase();
            if (!yn.equals("y")) break;
            System.out.print("Item name: "); String nm = sc.nextLine();
            System.out.print("Quantity: "); int qty = Integer.parseInt(sc.nextLine());
            System.out.print("Unit price: "); double up = Double.parseDouble(sc.nextLine());
            cons.addPrescription(new PrescriptionItem(nm, qty, up));
        }
        consultations.put(cid, cons);
        System.out.println("Consultation recorded:\n" + cons);
    }

    private static void generateInvoice() {
        System.out.println("-- Generate Invoice --");
        System.out.println("Available Consultations without invoice:");
        consultations.values().forEach(c -> {
            boolean hasInvoice = invoices.values().stream().anyMatch(inv -> inv.getConsultation().getId().equals(c.getId()));
            if (!hasInvoice) System.out.println(c.getId()+" - " + c.getAppointment().getPatient().getName()+" with " + c.getAppointment().getDoctor().getName());
        });
        System.out.print("Enter consultation id to invoice: "); String cid = sc.nextLine().trim();
        Consultation c = consultations.get(cid);
        if (c==null) { System.out.println("Consultation not found"); return; }

        boolean already = invoices.values().stream().anyMatch(inv -> inv.getConsultation().getId().equals(cid));
        if (already) { System.out.println("Invoice already exists for this consultation"); return; }

        String iid = IdGenerator.next("I");
        Invoice inv = new Invoice(iid, c, TAX_RATE);
        invoices.put(iid, inv);
        System.out.println("Invoice generated:\n" + inv);
    }

    private static void recordPayment() {
        System.out.println("-- Record Payment --");
        System.out.println("Unpaid Invoices:");
        invoices.values().stream().filter(inv -> !inv.isClosed()).forEach(inv -> System.out.println(inv.getId()+" - Total: " + String.format("%.2f", inv.getTotal())));
        System.out.print("Enter invoice id to pay: "); String iid = sc.nextLine().trim();
        Invoice inv = invoices.get(iid);
        if (inv==null) { System.out.println("Invoice not found"); return; }
        if (inv.isClosed()) { System.out.println("Invoice already paid"); return; }

        System.out.print("Enter payment amount: "); double amount = Double.parseDouble(sc.nextLine());
        // Business rule: Payment must settle the invoice before marking it closed.
        if (Math.abs(amount - inv.getTotal()) > 0.009) {
            System.out.println(String.format("Payment amount %.2f does not match invoice total %.2f. Payment rejected.", amount, inv.getTotal()));
            return;
        }
        String pid = IdGenerator.next("PAY");
        Payment p = new Payment(pid, inv, amount);
        payments.put(p.getId(), p);
        inv.markClosed();
        System.out.println("Payment recorded: " + p);
        System.out.println("Invoice marked PAID.");
    }

    private static void listAppointments() {
        System.out.println("-- All Appointments --");
        appointments.values().forEach(a->System.out.println(a));
    }

    private static void outstandingDuesReport() {
        System.out.println("-- Outstanding Dues Report --");
        invoices.values().stream().filter(inv -> !inv.isClosed()).forEach(inv -> {
            Consultation c = inv.getConsultation();
            System.out.println(String.format("Invoice %s | Patient: %s | Doctor: %s | Due: %.2f", inv.getId(), c.getAppointment().getPatient().getName(), c.getAppointment().getDoctor().getName(), inv.getTotal()));
        });
    }

}

/* End of file */
