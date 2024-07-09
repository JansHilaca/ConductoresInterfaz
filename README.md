# Interfaz Gráfica para Visualización de Datos de Carreras de Fórmula 1

Este proyecto es una interfaz gráfica desarrollada en Java con Swing para visualizar los datos de conductores de Fórmula 1 por año. Permite seleccionar un año específico y muestra en una tabla los datos de los conductores que participaron en las carreras de ese año, incluyendo su nombre, número de victorias, puntos totales y rango.

## Características

- Conexión a una base de datos PostgreSQL para obtener los datos de las carreras.
- Interfaz gráfica con un `JComboBox` para seleccionar el año de las carreras.
- Tabla (`JTable`) para mostrar los datos de los conductores, centrando el contenido de las celdas.

## Requisitos

- Java Development Kit (JDK) 8 o superior.
- PostgreSQL con una base de datos configurada y datos de Fórmula 1.
- Bibliotecas de Java Swing (incluidas en JDK).


2. Configuracion a la conexión a la base de datos PostgreSQL en el archivo `InterfazGrafica.java`:
    ```java
    private void connectDB() {
        try {
            String url = "jdbc:postgresql://localhost:5432/Formula1";
            String user = "postgres";
            String password = "merlina2004";
            conn = DriverManager.getConnection(url, user, password);
            System.out.println("Conexión establecida con PostgreSQL.");
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
    ```

3. Compila y ejecuta el programa:
    ```bash
    javac Deber_SegundoParcial/InterfazGrafica.java
    java Deber_SegundoParcial.InterfazGrafica
    ```

## Captura de Pantalla

![imagen](https://github.com/JansHilaca/ConductoresInterfaz/assets/168945853/0a2344af-c070-4656-a1fb-c347a21de6f9)

## Código Fuente

```java
package Deber_SegundoParcial;

import javax.swing.*;
import javax.swing.table.DefaultTableCellRenderer;
import javax.swing.table.DefaultTableModel;
import java.awt.*;
import java.awt.event.ActionEvent;
import java.io.FileWriter;
import java.io.IOException;
import java.sql.*;
import java.util.Vector;

public class InterfazGrafica {
    private Connection conn;
    private JFrame frame;
    private JComboBox<String> comboBox;
    private JTable driverTable;
    private DefaultTableModel driverTableModel;

    public InterfazGrafica() {
        // Establecer conexión a la base de datos PostgreSQL
        connectDB();

        // Crear la interfaz gráfica
        createGUI();
    }

    private void connectDB() {
        try {
            String url = "jdbc:postgresql://localhost:5432/Formula1";
            String user = "postgres";
            String password = "merlina2004";
            conn = DriverManager.getConnection(url, user, password);
            System.out.println("Conexión establecida con PostgreSQL.");
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    private void createGUI() {
        frame = new JFrame("Tabla de Conductores por Año de Carrera");
        frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        frame.setSize(800, 600);
        frame.setLayout(new BorderLayout());

        // Panel superior para el ComboBox
        JPanel topPanel = new JPanel();
        topPanel.setLayout(new FlowLayout());
        JLabel yearLabel = new JLabel("Año:");
        topPanel.add(yearLabel);

        // Combo box para seleccionar el año de carrera
        comboBox = new JComboBox<>();
        populateComboBox();
        comboBox.addActionListener(e -> updateTables());
        topPanel.add(comboBox);

        // Botón para exportar a CSV
        JButton exportButton = new JButton("Exportar a CSV");
        exportButton.addActionListener(this::exportToCSV);
        topPanel.add(exportButton);

        frame.add(topPanel, BorderLayout.NORTH);

        // Tabla para mostrar los datos de conductores
        driverTableModel = new DefaultTableModel();
        driverTable = new JTable(driverTableModel);
        JScrollPane driverScrollPane = new JScrollPane(driverTable);

        // Centrar el contenido de las celdas en la tabla
        DefaultTableCellRenderer centerRenderer = new DefaultTableCellRenderer();
        centerRenderer.setHorizontalAlignment(JLabel.CENTER);
        driverTable.setDefaultRenderer(Object.class, centerRenderer);

        frame.add(driverScrollPane, BorderLayout.CENTER);

        frame.setVisible(true);
    }

    private void populateComboBox() {
        try {
            Statement stmt = conn.createStatement();
            ResultSet rs = stmt.executeQuery("SELECT DISTINCT year FROM races ORDER BY year DESC");
            while (rs.next()) {
                comboBox.addItem(rs.getString("year"));
            }
            rs.close();
            stmt.close();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    private void updateTables() {
        updateDriverTable();
    }

    private void updateDriverTable() {
        try {
            String selectedYear = (String) comboBox.getSelectedItem();
            if (selectedYear != null) {
                String query = "SELECT d.forename || ' ' || d.surname AS driver_name, " +
                        "SUM(CASE WHEN ds.position = 1 THEN 1 ELSE 0 END) AS wins, " +
                        "SUM(ds.points) AS total_points, " +
                        "RANK() OVER (ORDER BY SUM(ds.points) DESC) AS rank " +
                        "FROM drivers d " +
                        "JOIN driver_standings ds ON d.driver_id = ds.driver_id " +
                        "JOIN races r ON ds.race_id = r.race_id " +
                        "WHERE r.year = ? " +
                        "GROUP BY d.driver_id " +
                        "ORDER BY rank";

                PreparedStatement pstmt = conn.prepareStatement(query);
                pstmt.setInt(1, Integer.parseInt(selectedYear));
                ResultSet rs = pstmt.executeQuery();

                // Obtener columnas
                Vector<String> columnNames = new Vector<>();
                columnNames.add("Driver Name");
                columnNames.add("Wins");
                columnNames.add("Total Points");
                columnNames.add("Rank");

                // Obtener filas
                Vector<Vector<Object>> data = new Vector<>();
                while (rs.next()) {
                    Vector<Object> row = new Vector<>();
                    row.add(rs.getString("driver_name"));
                    row.add(rs.getInt("wins"));
                    row.add(rs.getDouble("total_points"));
                    row.add(rs.getInt("rank"));
                    data.add(row);
                }

                // Actualizar modelo de la tabla
                driverTableModel.setDataVector(data, columnNames);

                rs.close();
                pstmt.close();
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    private void exportToCSV(ActionEvent e) {
        try (FileWriter csvWriter = new FileWriter("drivers.csv")) {
            for (int i = 0; i < driverTableModel.getColumnCount(); i++) {
                csvWriter.write(driverTableModel.getColumnName(i) + ",");
            }
            csvWriter.write("\n");

            for (int i = 0; i < driverTableModel.getRowCount(); i++) {
                for (int j = 0; j < driverTableModel.getColumnCount(); j++) {
                    csvWriter.write(driverTableModel.getValueAt(i, j).toString() + ",");
                }
                csvWriter.write("\n");
            }

            JOptionPane.showMessageDialog(frame, "Datos exportados correctamente a drivers.csv");
        } catch (IOException ex) {
            JOptionPane.showMessageDialog(frame, "Error al exportar los datos: " + ex.getMessage());
        }
    }

    public static void main(String[] args) {
        SwingUtilities.invokeLater(InterfazGrafica::new);
    }
}

