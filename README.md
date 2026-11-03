# BusReservationSystem

import java.sql.*;
import java.util.Scanner;

public class BusReservationSystem {
    private static final String BASE_URL = "jdbc:mysql://localhost:3306/";
    private static final String DB_NAME = "bus_reservation_db";
    private static final String USER = "root";
    private static final String PASSWORD = "password";

    public static void main(String[] args) {
        initializeDatabase();

        Scanner scanner = new Scanner(System.in);
        String connectionUrl = BASE_URL + DB_NAME;

        try (Connection conn = DriverManager.getConnection(connectionUrl, USER, PASSWORD)) {
            conn.setAutoCommit(false);

            while (true) {
                System.out.println("\n===== BUS RESERVATION SYSTEM =====");
                System.out.println("1. Search Buses");
                System.out.println("2. Book Seat");
                System.out.println("3. Cancel Ticket");
                System.out.println("4. Exit");
                System.out.print("Choose an option: ");

                int choice = scanner.nextInt();
                switch (choice) {
                    case 1:
                        searchBuses(conn, scanner);
                        break;
                    case 2:
                        bookSeat(conn, scanner);
                        break;
                    case 3:
                        cancelTicket(conn, scanner);
                        break;
                    case 4:
                        System.out.println("Exiting... Thank you!");
                        return;
                    default:
                        System.out.println("Invalid choice. Try again.");
                }
            }
        } catch (SQLException e) {
            System.err.println("Database error occurred: " + e.getMessage());
        }
    }
    private static void initializeDatabase() {
        try (Connection conn = DriverManager.getConnection(BASE_URL, USER, PASSWORD);
             Statement stmt = conn.createStatement()) {
            
            // Create database
            stmt.executeUpdate("CREATE DATABASE IF NOT EXISTS " + DB_NAME);
            stmt.executeUpdate("USE " + DB_NAME);

            // Create buses table
            stmt.executeUpdate("CREATE TABLE IF NOT EXISTS buses (" +
                    "bus_id INT AUTO_INCREMENT PRIMARY KEY, " +
                    "bus_number VARCHAR(20) NOT NULL UNIQUE, " +
                    "source VARCHAR(50) NOT NULL, " +
                    "destination VARCHAR(50) NOT NULL, " +
                    "total_seats INT NOT NULL, " +
                    "available_seats INT NOT NULL, " +
                    "fare DECIMAL(10, 2) NOT NULL)");

            // Create passengers table
            stmt.executeUpdate("CREATE TABLE IF NOT EXISTS passengers (" +
                    "passenger_id INT AUTO_INCREMENT PRIMARY KEY, " +
                    "name VARCHAR(100) NOT NULL, " +
                    "age INT NOT NULL, " +
                    "gender VARCHAR(10) NOT NULL, " +
                    "phone VARCHAR(15) NOT NULL)");

            // Create bookings table
            stmt.executeUpdate("CREATE TABLE IF NOT EXISTS bookings (" +
                    "booking_id INT AUTO_INCREMENT PRIMARY KEY, " +
                    "bus_id INT NOT NULL, " +
                    "passenger_id INT NOT NULL, " +
                    "seat_number INT NOT NULL, " +
                    "booking_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP, " +
                    "status VARCHAR(20) DEFAULT 'CONFIRMED', " +
                    "FOREIGN KEY (bus_id) REFERENCES buses(bus_id), " +
                    "FOREIGN KEY (passenger_id) REFERENCES passengers(passenger_id))");

            ResultSet rs = stmt.executeQuery("SELECT COUNT(*) FROM buses");
            if (rs.next() && rs.getInt(1) == 0) {
                stmt.executeUpdate("INSERT INTO buses (bus_number, source, destination, total_seats, available_seats, fare) VALUES " +
                        "('TN-20-A-1001', 'Chennai', 'Bangalore', 40, 40, 750.00), " +
                        "('TN-20-B-2002', 'Chennai', 'Madurai', 35, 35, 600.00)");
                System.out.println("Sample bus routes initialized successfully.");
            }

        } catch (SQLException e) {
            System.err.println("Failed to initialize database: " + e.getMessage());
            System.exit(1);
        }
    }
    private static void searchBuses(Connection conn, Scanner scanner) throws SQLException {
        scanner.nextLine(); 
        System.out.print("Enter Source: ");
        String source = scanner.nextLine();
        System.out.print("Enter Destination: ");
        String dest = scanner.nextLine();

        String query = "SELECT * FROM buses WHERE source LIKE ? AND destination LIKE ?";
        try (PreparedStatement stmt = conn.prepareStatement(query)) {
            stmt.setString(1, "%" + source + "%");
            stmt.setString(2, "%" + dest + "%");
            ResultSet rs = stmt.executeQuery();

            System.out.println("\n--- Available Buses ---");
            boolean found = false;
            while (rs.next()) {
                found = true;
                System.out.printf("Bus ID: %d | Bus No: %s | Route: %s -> %s | Available Seats: %d/%d | Fare: %.2f\n",
                        rs.getInt("bus_id"),
                        rs.getString("bus_number"),
                        rs.getString("source"),
                        rs.getString("destination"),
                        rs.getInt("available_seats"),
                        rs.getInt("total_seats"),
                        rs.getDouble("fare"));
            }
            if (!found) {
                System.out.println("No buses found for this route.");
            }
        }
    } 
    private static void bookSeat(Connection conn, Scanner scanner) {
        try {
            System.out.print("Enter Bus ID: ");
            int busId = scanner.nextInt();
            scanner.nextLine(); 

            System.out.print("Enter Passenger Name: ");
            String name = scanner.nextLine();
            System.out.print("Enter Age: ");
            int age = scanner.nextInt();
            scanner.nextLine();
            
            System.out.print("Enter Gender: ");
            String gender = scanner.nextLine();
            System.out.print("Enter Phone: ");
            String phone = scanner.nextLine();
            System.out.print("Enter Seat Number to Book: ");
            int seatNumber = scanner.nextInt();

            String checkBusSQL = "SELECT available_seats FROM buses WHERE bus_id = ? FOR UPDATE";
            try (PreparedStatement checkStmt = conn.prepareStatement(checkBusSQL)) {
                checkStmt.setInt(1, busId);
                ResultSet rs = checkStmt.executeQuery();
                if (!rs.next()) {
                    System.out.println("Bus not found.");
                    conn.rollback();
                    return;
                }
                int availableSeats = rs.getInt("available_seats");
                if (availableSeats <= 0) {
                    System.out.println("Sorry, no available seats on this bus.");
                    conn.rollback();
                    return;
                }
            }

            String checkSeatSQL = "SELECT booking_id FROM bookings WHERE bus_id = ? AND seat_number = ? AND status = 'CONFIRMED'";
            try (PreparedStatement seatStmt = conn.prepareStatement(checkSeatSQL)) {
                seatStmt.setInt(1, busId);
                seatStmt.setInt(2, seatNumber);
                ResultSet rsSeat = seatStmt.executeQuery();
                if (rsSeat.next()) {
                    System.out.println("Seat number " + seatNumber + " is already booked.");
                    conn.rollback();
                    return;
                }
            }

            String insertPassengerSQL = "INSERT INTO passengers (name, age, gender, phone) VALUES (?, ?, ?, ?)";
            int passengerId;
            try (PreparedStatement pStmt = conn.prepareStatement(insertPassengerSQL, Statement.RETURN_GENERATED_KEYS)) {
                pStmt.setString(1, name);
                pStmt.setInt(2, age);
                pStmt.setString(3, gender);
                pStmt.setString(4, phone);
                pStmt.executeUpdate();

                ResultSet genKeys = pStmt.getGeneratedKeys();
                if (genKeys.next()) {
                    passengerId = genKeys.getInt(1);
                } else {
                    throw new SQLException("Failed to retrieve passenger ID.");
                }
            }

            String insertBookingSQL = "INSERT INTO bookings (bus_id, passenger_id, seat_number, status) VALUES (?, ?, ?, 'CONFIRMED')";
            try (PreparedStatement bStmt = conn.prepareStatement(insertBookingSQL)) {
                bStmt.setInt(1, busId);
                bStmt.setInt(2, passengerId);
                bStmt.setInt(3, seatNumber);
                bStmt.executeUpdate();
            }

            String updateBusSQL = "UPDATE buses SET available_seats = available_seats - 1 WHERE bus_id = ?";
            try (PreparedStatement uStmt = conn.prepareStatement(updateBusSQL)) {
            uStmt.setInt(1, busId);
            uStmt.executeUpdate();
            }
            conn.commit();
            System.out.println("Seat booked successfully! Transaction committed.");
            } catch (SQLException e) {
            try {
            conn.rollback();
            System.out.println("Booking failed. Transaction rolled back: " + e.getMessage());
            } catch (SQLException rollbackEx) {
            rollbackEx.printStackTrace();
            }
            }
            }
            private static void cancelTicket(Connection conn, Scanner scanner) {
try {
System.out.print("Enter Booking ID to cancel: ");
int bookingId = scanner.nextInt();

String getBookingSQL = "SELECT bus_id, status FROM bookings WHERE booking_id = ?";
int busId = -1;
String status = "";
try (PreparedStatement getStmt = conn.prepareStatement(getBookingSQL)) {
getStmt.setInt(1, bookingId);
ResultSet rs = getStmt.executeQuery();
if (!rs.next()) {
System.out.println("Booking ID not found.");
conn.rollback();
return;
}
busId = rs.getInt("bus_id");
status = rs.getString("status");
}
if ("CANCELLED".equals(status)) {
System.out.println("This ticket is already cancelled.");
conn.rollback();
return;
}

String updateBookingSQL = "UPDATE bookings SET status = 'CANCELLED' WHERE booking_id = ?";
try (PreparedStatement ubStmt = conn.prepareStatement(updateBookingSQL)) {
ubStmt.setInt(1, bookingId);
ubStmt.executeUpdate();
}

String updateBusSQL = "UPDATE buses SET available_seats = available_seats + 1 WHERE bus_id = ?";
try (PreparedStatement ubBusStmt = conn.prepareStatement(updateBusSQL)) {
ubBusStmt.setInt(1, busId);
ubBusStmt.executeUpdate();
}

conn.commit();
System.out.println("Ticket cancelled successfully! Available seat restored.");
} 
catch (SQLException e) {
try {
conn.rollback();
System.out.println("Cancellation failed. Rolled back: " + e.getMessage());
} catch (SQLException rollbackEx) {
rollbackEx.printStackTrace();
}
}
}
}
