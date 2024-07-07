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

## Configuración de la Base de Datos

1. Asegúrate de tener PostgreSQL instalado y en funcionamiento.
2. Crea una base de datos llamada `Formula1`.
3. Crea las tablas necesarias y puebla los datos relacionados con las carreras de Fórmula 1 (drivers, races, driver_standings, etc.).

## Uso

1. Clona este repositorio:
    ```bash
    git clone https://github.com/tu-usuario/interfaz-grafica-f1.git
    cd interfaz-grafica-f1
    ```

2. Configura la conexión a la base de datos PostgreSQL en el archivo `InterfazGrafica.java`:
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

![imagen](https://github.com/JansHilaca/ConductoresInterfaz/assets/168945853/82d7ea64-8c87-467a-8a6f-e8c43e20e474)

## Código Fuente

```java
package Deber_SegundoParcial;

import javax.swing.*;
import javax.swing.table.DefaultTableCellRenderer;
import javax.swing.table.DefaultTableModel;
import java.awt.*;
import java.sql.*;
import java.util.Vector;

public class InterfazGrafica {
    private Connection conn;
    private JFrame frame;
    private JComboBox<String> comboBox;
    private JTable table;
    private DefaultTableModel tableModel;

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
        frame = new JFrame("Tabla de Drivers por Año de Carrera");
        frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        frame.setSize(600, 400);
        frame.setLayout(new BorderLayout());

        // Panel superior para el ComboBox
        JPanel topPanel = new JPanel();
        topPanel.setLayout(new FlowLayout());
        JLabel yearLabel = new JLabel("Año:");
        topPanel.add(yearLabel);

        // Combo box para seleccionar el año de carrera
        comboBox = new JComboBox<>();
        populateComboBox();
        comboBox.addActionListener(e -> {
            // Cuando se seleccione un año, actualizar la tabla de corredores
            updateTable();
        });
        topPanel.add(comboBox);

        frame.add(topPanel, BorderLayout.NORTH);

        // Tabla para mostrar los datos de corredores y carreras
        tableModel = new DefaultTableModel();
        table = new JTable(tableModel);
        JScrollPane scrollPane = new JScrollPane(table);

        // Centrar el contenido de las celdas
        DefaultTableCellRenderer centerRenderer = new DefaultTableCellRenderer();
        centerRenderer.setHorizontalAlignment(JLabel.CENTER);
        table.setDefaultRenderer(Object.class, centerRenderer);

        frame.add(scrollPane, BorderLayout.CENTER);

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

    private void updateTable() {
        try {
            String selectedYear = (String) comboBox.getSelectedItem();
            if (selectedYear != null) {
                // Consulta para obtener los corredores que participaron en las carreras del año seleccionado
                String query = "SELECT DISTINCT ON (d.driver_id) d.driver_id, d.forename, d.surname, d.dob, d.nationality, " +
                        "(SELECT COUNT(*) FROM driver_standings ds INNER JOIN races r ON ds.race_id = r.race_id " +
                        "WHERE ds.driver_id = d.driver_id AND r.year = ? AND ds.position = 1) AS carreras_ganadas, " +
                        "(SELECT COUNT(*) FROM driver_standings ds INNER JOIN races r ON ds.race_id = r.race_id " +
                        "WHERE ds.driver_id = d.driver_id AND r.year = ?) AS num_races " +
                        "FROM drivers d " +
                        "JOIN driver_standings ds ON d.driver_id = ds.driver_id " +
                        "JOIN races r ON ds.race_id = r.race_id " +
                        "WHERE r.year = ? " +
                        "ORDER BY d.driver_id, r.date";

                PreparedStatement pstmt = conn.prepareStatement(query);
                pstmt.setInt(1, Integer.parseInt(selectedYear));
                pstmt.setInt(2, Integer.parseInt(selectedYear));
                pstmt.setInt(3, Integer.parseInt(selectedYear));
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
                    row.add(rs.getString("forename") + " " + rs.getString("surname"));
                    row.add(rs.getInt("carreras_ganadas"));
                    row.add(rs.getInt("num_races"));
                    row.add(rs.getString("dob"));  // Adjust as per actual rank calculation if needed
                    data.add(row);
                }

                // Actualizar modelo de la tabla
                tableModel.setDataVector(data, columnNames);

                // Centrar el contenido de las celdas
                DefaultTableCellRenderer centerRenderer = new DefaultTableCellRenderer();
                centerRenderer.setHorizontalAlignment(JLabel.CENTER);
                table.setDefaultRenderer(Object.class, centerRenderer);

                rs.close();
                pstmt.close();
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        SwingUtilities.invokeLater(InterfazGrafica::new);
    }
}
