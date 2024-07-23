# Constructores_Puntos

## Captura

![imagen](https://github.com/user-attachments/assets/9dac93d6-095d-46ca-82a1-df35a88358d7)


### Código Fuente

```java
package Interfaz_Constructores;

import javax.swing.*;
import javax.swing.table.DefaultTableCellRenderer;
import javax.swing.table.DefaultTableModel;
import java.awt.*;
import java.sql.*;
import java.util.Vector;

public class Main {
    private Connection conn;
    private JFrame frame;
    private JComboBox<String> comboBox;
    private JTable table;
    private DefaultTableModel tableModel;
    private JProgressBar progressBar;
    private JButton refreshButton;
    private JButton exitButton;

    public Main() {
        // Establecer conexión con la base de datos PostgreSQL
        connectDB();
        
        // Crear la interfaz gráfica
        createGUI();
    }

    private void connectDB() {
        try {
            String url = "jdbc:postgresql://localhost:5432/Corredores";
            String user = "postgres";
            String password = "tu_contraseña";
            conn = DriverManager.getConnection(url, user, password);
            System.out.println("Connected to PostgreSQL.");
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    private void createGUI() {
        frame = new JFrame("Top Constructors by Points");
        frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        frame.setSize(800, 600);
        frame.setLayout(new BorderLayout(10, 10));

        // Panel superior para la selección de año y botones
        JPanel topPanel = new JPanel();
        topPanel.setLayout(new FlowLayout());

        // Combo box para seleccionar el año de la carrera
        comboBox = new JComboBox<>();
        populateComboBox();
        comboBox.addActionListener(e -> updateTableInBackground());

        // Botón para refrescar manualmente los datos
        refreshButton = new JButton("Refresh");
        refreshButton.addActionListener(e -> updateTableInBackground());

        // Botón para salir de la aplicación
        exitButton = new JButton("Exit");
        exitButton.addActionListener(e -> System.exit(0));

        topPanel.add(new JLabel("Select Year:"));
        topPanel.add(comboBox);
        topPanel.add(refreshButton);
        topPanel.add(exitButton);

        // Barra de progreso para mostrar mientras la tabla se está cargando
        progressBar = new JProgressBar();
        progressBar.setStringPainted(true);

        // Tabla para mostrar los datos de puntos de los constructores
        tableModel = new DefaultTableModel();
        table = new JTable(tableModel);
        JScrollPane scrollPane = new JScrollPane(table);

        // Centrar el contenido de las celdas
        DefaultTableCellRenderer centerRenderer = new DefaultTableCellRenderer();
        centerRenderer.setHorizontalAlignment(JLabel.CENTER);
        table.setDefaultRenderer(Object.class, centerRenderer);

        // Añadir componentes al frame
        frame.add(topPanel, BorderLayout.NORTH);
        frame.add(scrollPane, BorderLayout.CENTER);
        frame.add(progressBar, BorderLayout.SOUTH);

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

    private void updateTableInBackground() {
        String selectedYear = (String) comboBox.getSelectedItem();
        if (selectedYear != null) {
            // Crear un SwingWorker para ejecutar la consulta en segundo plano
            SwingWorker<Void, Void> worker = new SwingWorker<Void, Void>() {
                @Override
                protected Void doInBackground() throws Exception {
                    try {
                        // Consulta para obtener los constructores y sus puntos totales para el año seleccionado
                        String query = "SELECT c.name AS constructor_name, SUM(cs.points) AS total_points " +
                                "FROM constructors c " +
                                "JOIN constructor_standings cs ON c.constructor_id = cs.constructor_id " +
                                "JOIN races r ON cs.race_id = r.race_id " +
                                "WHERE r.year = ? " +
                                "GROUP BY constructor_name " +
                                "ORDER BY total_points DESC";

                        PreparedStatement pstmt = conn.prepareStatement(query);
                        pstmt.setInt(1, Integer.parseInt(selectedYear));
                        ResultSet rs = pstmt.executeQuery();

                        // Obtener columnas
                        Vector<String> columnNames = new Vector<>();
                        columnNames.add("Constructor Name");
                        columnNames.add("Total Points");

                        // Obtener filas
                        Vector<Vector<Object>> data = new Vector<>();
                        while (rs.next()) {
                            Vector<Object> row = new Vector<>();
                            row.add(rs.getString("constructor_name"));
                            row.add(rs.getDouble("total_points"));
                            data.add(row);
                        }

                        // Actualizar el modelo de la tabla en el hilo de eventos de Swing
                        SwingUtilities.invokeLater(() -> {
                            tableModel.setDataVector(data, columnNames);
                            progressBar.setValue(100); // Completar la barra de progreso
                        });

                        rs.close();
                        pstmt.close();
                    } catch (SQLException e) {
                        e.printStackTrace();
                    }
                    return null;
                }

                @Override
                protected void done() {
                    // Acciones adicionales después de completar la carga de datos
                    progressBar.setIndeterminate(false); // Detener el estado indeterminado
                }
            };

            // Iniciar el SwingWorker y mostrar la barra de progreso
            progressBar.setValue(0); // Restablecer la barra de progreso
            progressBar.setIndeterminate(true); // Mostrar una barra de progreso indeterminada
            worker.execute();
        }
    }

    public static void main(String[] args) {
        SwingUtilities.invokeLater(Main::new);
    }
}

