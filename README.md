# Impresora
import javax.imageio.ImageIO;
import javax.imageio.ImageReader;
import javax.imageio.metadata.IIOMetadata;
import javax.imageio.metadata.IIOMetadataNode;
import javax.imageio.stream.ImageInputStream;
import javax.print.PrintService;
import javax.print.PrintServiceLookup;
import javax.print.attribute.HashPrintRequestAttributeSet;
import javax.print.attribute.PrintRequestAttributeSet;
import javax.print.attribute.standard.Chromaticity;
import javax.print.attribute.standard.Copies;
import javax.print.attribute.standard.MediaSizeName;
import javax.print.attribute.standard.OrientationRequested;
import javax.print.attribute.standard.PrintQuality;
import javax.print.attribute.standard.Sides;
import javax.swing.*;
import javax.swing.filechooser.FileNameExtensionFilter;
import org.w3c.dom.NodeList;
import java.awt.*;
import java.awt.datatransfer.DataFlavor;
import java.awt.geom.AffineTransform;
import java.awt.image.BufferedImage;
import java.awt.image.ConvolveOp;
import java.awt.image.Kernel;
import java.awt.image.Raster;
import java.awt.print.PageFormat;
import java.awt.print.Paper;
import java.awt.print.Printable;
import java.awt.print.PrinterException;
import java.awt.print.PrinterJob;
import java.io.File;
import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.HashSet;
import java.util.Iterator;
import java.util.List;
import java.util.Set;

/**
 * ImpressoraPro - impressora em Java com:
 *  - fila de vários arquivos (imagens, PDF, textos), arrastar e soltar;
 *  - escolha da impressora, cópias, cores, qualidade, frente e verso;
 *  - papel (A4, Carta, A3, A5, Ofício), orientação, margens;
 *  - escala: 100% (tamanho real), ajustar, preencher, personalizada;
 *  - imagem grande dividida em várias folhas, centralização;
 *  - prévia da folha (WYSIWYG) e melhoria de resolução (300/450/600 DPI + nitidez);
 *  - leitura de imagens complexas (CMYK, transparência, rotação EXIF de celular).
 *
 * Compilar: javac ImpressoraPro.java
 * Executar: java ImpressoraPro
 *
 * PDF (opcional): coloque o pdfbox-app-x.y.z.jar na pasta e execute
 *   Windows: java -cp ".;pdfbox-app-x.y.z.jar" ImpressoraPro
 *   Linux/Mac: java -cp ".:pdfbox-app-x.y.z.jar" ImpressoraPro
 */
public class ImpressoraPro extends JFrame {

    enum Tipo { IMAGEM, PDF, TEXTO }

    static class Item {
        File arquivo;
        Tipo tipo;
        String status = "Aguardando";
        BufferedImage img, mini;          // imagem
        int dpiArquivo;                   // DPI gravado no arquivo (0 = desconhecido)
        Object pdfDoc;                    // PDF (PDFBox via reflexão)
        int paginasPdf;
        double[][] tamPdf;
        BufferedImage miniPdf;
        List<String> linhas;              // texto
        String cacheChave = "";
        BufferedImage cacheImg;

        @Override
        public String toString() {
            String t = tipo == Tipo.IMAGEM ? "IMG" : tipo == Tipo.PDF ? "PDF" : "TXT";
            return "[" + t + "] " + arquivo.getName() + "   —   " + status;
        }
    }

    static class Colocacao {
        double x, y, w, h;
        int colunas = 1, linhas = 1;
    }

    private static final String[] PAPEIS = {"A4", "Carta", "A3", "A5", "Ofício"};
    private static final double[][] PAPEL_PT = {
            {595.28, 841.89}, {612, 792}, {841.89, 1190.55}, {419.53, 595.28}, {612, 1008}};
    private static final MediaSizeName[] PAPEL_MEDIA = {
            MediaSizeName.ISO_A4, MediaSizeName.NA_LETTER, MediaSizeName.ISO_A3,
            MediaSizeName.ISO_A5, MediaSizeName.NA_LEGAL};
    private static final int[] DPIS = {150, 300, 450, 600};
    private static final Set<String> TEXTOS = new HashSet<>(Arrays.asList(
            "txt", "log", "java", "csv", "md", "xml", "json", "html", "htm", "ini", "bat", "sql", "py", "js", "css"));

    // ---- Componentes
    private final DefaultListModel<Item> modelo = new DefaultListModel<>();
    private final JList<Item> lista = new JList<>(modelo);
    private final JComboBox<PrintService> cmbImpressora;
    private final JSpinner spnCopias = new JSpinner(new SpinnerNumberModel(1, 1, 99, 1));
    private final JComboBox<String> cmbCor = new JComboBox<>(new String[]{"Colorido", "Preto e branco"});
    private final JComboBox<String> cmbQualidade = new JComboBox<>(new String[]{
            "Rascunho (150 DPI)", "Normal (300 DPI)", "Alta (450 DPI)", "Máxima (600 DPI)"});
    private final JComboBox<String> cmbLados = new JComboBox<>(new String[]{
            "Um lado", "Frente e verso (borda longa)", "Frente e verso (borda curta)"});
    private final JComboBox<String> cmbPapel = new JComboBox<>(PAPEIS);
    private final JComboBox<String> cmbOrient = new JComboBox<>(new String[]{"Automática", "Retrato", "Paisagem"});
    private final JSpinner spnMargem = new JSpinner(new SpinnerNumberModel(5, 0, 50, 1));
    private final JRadioButton rb100 = new JRadioButton("100% (tamanho real)", true);
    private final JRadioButton rbAjustar = new JRadioButton("Ajustar à página");
    private final JRadioButton rbPreencher = new JRadioButton("Preencher a página (corta as sobras)");
    private final JRadioButton rbPers = new JRadioButton("Personalizado:");
    private final JSpinner spnPct = new JSpinner(new SpinnerNumberModel(100, 5, 800, 5));
    private final JCheckBox chkDividir = new JCheckBox("Dividir imagem grande em várias folhas", true);
    private final JCheckBox chkCentro = new JCheckBox("Centralizar na folha", true);
    private final JCheckBox chkDpiArq = new JCheckBox("Usar o DPI gravado no arquivo", true);
    private final JSpinner spnDpi = new JSpinner(new SpinnerNumberModel(96, 36, 1200, 1));
    private final JCheckBox chkNitidez = new JCheckBox("Aumentar nitidez", true);
    private final JSpinner spnFonte = new JSpinner(new SpinnerNumberModel(10, 6, 40, 1));
    private final JCheckBox chkDialogo = new JCheckBox("Mostrar diálogo do sistema antes de imprimir");
    private final JLabel lblInfo = new JLabel(" ", SwingConstants.CENTER);
    private final JLabel lblStatus = new JLabel("Arraste arquivos para a fila ou use os botões.");
    private final JProgressBar barra = new JProgressBar(0, 100);
    private final JButton btnImprimir = new JButton("IMPRIMIR FILA");
    private final JButton btnCancelar = new JButton("Cancelar");
    private final Previa previa = new Previa();

    private File ultimaPasta = new File(System.getProperty("user.home"));
    private PrintRequestAttributeSet attrsDialogo;
    private volatile boolean cancelar;

    public ImpressoraPro() {
        super("ImpressoraPro - Imprimir 100% com Java");
        setDefaultCloseOperation(EXIT_ON_CLOSE);
        setSize(1180, 740);
        setLocationRelativeTo(null);

        // Impressoras instaladas
        PrintService[] servicos = PrintServiceLookup.lookupPrintServices(null, null);
        PrintService padrao = PrintServiceLookup.lookupDefaultPrintService();
        cmbImpressora = new JComboBox<>(servicos);
        if (padrao != null) cmbImpressora.setSelectedItem(padrao);
        cmbImpressora.setRenderer(new DefaultListCellRenderer() {
            @Override
            public Component getListCellRendererComponent(JList<?> l, Object v, int i, boolean s, boolean f) {
                Object txt = v instanceof PrintService ? ((PrintService) v).getName() : v;
                return super.getListCellRendererComponent(l, txt, i, s, f);
            }
        });
        cmbQualidade.setSelectedIndex(2);

        ButtonGroup bg = new ButtonGroup();
        bg.add(rb100);
        bg.add(rbAjustar);
        bg.add(rbPreencher);
        bg.add(rbPers);

        // ---- Esquerda: fila
        lista.setSelectionMode(ListSelectionModel.SINGLE_SELECTION);
        lista.addListSelectionListener(e -> {
            if (!e.getValueIsAdjusting()) {
                atualizarInfo();
                previa.repaint();
            }
        });
        habilitarArrastar();
        JScrollPane spLista = new JScrollPane(lista);
        spLista.setBorder(BorderFactory.createTitledBorder("Fila de impressão (arraste arquivos aqui)"));

        JButton bArq = new JButton("Adicionar arquivos");
        bArq.addActionListener(e -> escolherArquivos());
        JButton bPasta = new JButton("Adicionar pasta");
        bPasta.addActionListener(e -> escolherPasta());
        JButton bRem = new JButton("Remover");
        bRem.addActionListener(e -> {
            int i = lista.getSelectedIndex();
            if (i >= 0) modelo.remove(i);
            previa.repaint();
        });
        JButton bLimpar = new JButton("Limpar tudo");
        bLimpar.addActionListener(e -> {
            modelo.clear();
            previa.repaint();
        });
        JButton bSobe = new JButton("Subir");
        bSobe.addActionListener(e -> mover(-1));
        JButton bDesce = new JButton("Descer");
        bDesce.addActionListener(e -> mover(1));
        JPanel botoesFila = new JPanel(new GridLayout(3, 2, 6, 6));
        for (JButton b : new JButton[]{bArq, bPasta, bRem, bLimpar, bSobe, bDesce}) botoesFila.add(b);

        JPanel esquerda = new JPanel(new BorderLayout(0, 6));
        esquerda.setPreferredSize(new Dimension(300, 0));
        esquerda.add(spLista, BorderLayout.CENTER);
        esquerda.add(botoesFila, BorderLayout.SOUTH);

        // ---- Centro: prévia
        JPanel centro = new JPanel(new BorderLayout());
        centro.add(previa, BorderLayout.CENTER);
        lblInfo.setBorder(BorderFactory.createEmptyBorder(6, 6, 6, 6));
        centro.add(lblInfo, BorderLayout.SOUTH);

        // ---- Direita: configurações
        JPanel cfg = new JPanel();
        cfg.setLayout(new BoxLayout(cfg, BoxLayout.Y_AXIS));
        cfg.add(secao("Impressora",
                linha("Impressora:", cmbImpressora), linha("Cópias:", spnCopias), linha("Cores:", cmbCor),
                linha("Qualidade:", cmbQualidade), linha("Lados:", cmbLados)));
        cfg.add(secao("Página",
                linha("Papel:", cmbPapel), linha("Orientação:", cmbOrient), linha("Margem (mm):", spnMargem)));
        JPanel linhaPers = new JPanel(new FlowLayout(FlowLayout.LEFT, 4, 0));
        linhaPers.add(rbPers);
        linhaPers.add(spnPct);
        linhaPers.add(new JLabel("%"));
        cfg.add(secao("Escala", rb100, rbAjustar, rbPreencher, linhaPers, chkDividir, chkCentro));
        cfg.add(secao("Imagem e texto",
                chkDpiArq, linha("DPI padrão:", spnDpi), chkNitidez, linha("Fonte do texto:", spnFonte)));
        cfg.add(secao("Avançado", chkDialogo));
        JScrollPane spCfg = new JScrollPane(cfg);
        spCfg.setBorder(null);
        spCfg.setPreferredSize(new Dimension(360, 0));
        spCfg.getVerticalScrollBar().setUnitIncrement(16);

        // ---- Rodapé
        btnImprimir.setFont(btnImprimir.getFont().deriveFont(Font.BOLD, 16f));
        btnImprimir.setBackground(new Color(46, 160, 67));
        btnImprimir.setForeground(Color.WHITE);
        btnImprimir.setOpaque(true);
        btnImprimir.setBorderPainted(false);
        btnImprimir.setPreferredSize(new Dimension(220, 46));
        btnImprimir.addActionListener(e -> imprimirFila());
        btnCancelar.setEnabled(false);
        btnCancelar.addActionListener(e -> cancelar = true);
        barra.setStringPainted(true);
        JPanel sul = new JPanel(new BorderLayout(10, 0));
        sul.setBorder(BorderFactory.createEmptyBorder(6, 0, 0, 0));
        JPanel meio = new JPanel(new GridLayout(2, 1, 0, 2));
        meio.add(lblStatus);
        meio.add(barra);
        sul.add(meio, BorderLayout.CENTER);
        JPanel dir = new JPanel(new FlowLayout(FlowLayout.RIGHT, 6, 0));
        dir.add(btnCancelar);
        dir.add(btnImprimir);
        sul.add(dir, BorderLayout.EAST);

        JPanel raiz = new JPanel(new BorderLayout(10, 10));
        raiz.setBorder(BorderFactory.createEmptyBorder(10, 10, 10, 10));
        raiz.add(esquerda, BorderLayout.WEST);
        raiz.add(centro, BorderLayout.CENTER);
        raiz.add(spCfg, BorderLayout.EAST);
        raiz.add(sul, BorderLayout.SOUTH);
        setContentPane(raiz);

        ouvir(cmbCor, cmbQualidade, cmbLados, cmbPapel, cmbOrient, spnMargem, rb100, rbAjustar, rbPreencher,
                rbPers, spnPct, chkDividir, chkCentro, chkDpiArq, spnDpi, chkNitidez, spnFonte);
    }

    // =====================================================================
    //  Interface auxiliar
    // =====================================================================

    private JPanel linha(String rotulo, JComponent comp) {
        JPanel p = new JPanel(new BorderLayout(6, 0));
        JLabel l = new JLabel(rotulo);
        l.setPreferredSize(new Dimension(95, 24));
        p.add(l, BorderLayout.WEST);
        p.add(comp, BorderLayout.CENTER);
        return p;
    }

    private JPanel secao(String titulo, JComponent... filhos) {
        JPanel p = new JPanel(new GridLayout(0, 1, 0, 4));
        p.setBorder(BorderFactory.createTitledBorder(titulo));
        for (JComponent c : filhos) p.add(c);
        p.setAlignmentX(Component.LEFT_ALIGNMENT);
        p.setMaximumSize(new Dimension(Integer.MAX_VALUE, p.getPreferredSize().height));
        return p;
    }

    private void ouvir(JComponent... comps) {
        for (JComponent c : comps) {
            if (c instanceof JSpinner) ((JSpinner) c).addChangeListener(e -> previa.repaint());
            else if (c instanceof JComboBox) ((JComboBox<?>) c).addActionListener(e -> previa.repaint());
            else if (c instanceof AbstractButton) ((AbstractButton) c).addActionListener(e -> previa.repaint());
        }
    }

    private void mover(int delta) {
        int i = lista.getSelectedIndex(), j = i + delta;
        if (i < 0 || j < 0 || j >= modelo.size()) return;
        Item a = modelo.get(i);
        modelo.set(i, modelo.get(j));
        modelo.set(j, a);
        lista.setSelectedIndex(j);
    }

    @SuppressWarnings("unchecked")
    private void habilitarArrastar() {
        lista.setTransferHandler(new TransferHandler() {
            @Override
            public boolean canImport(TransferSupport s) {
                return s.isDataFlavorSupported(DataFlavor.javaFileListFlavor);
            }

            @Override
            public boolean importData(TransferSupport s) {
                try {
                    List<File> fs = (List<File>) s.getTransferable().getTransferData(DataFlavor.javaFileListFlavor);
                    adicionar(fs);
                    return true;
                } catch (Exception e) {
                    return false;
                }
            }
        });
    }

    private void atualizarInfo() {
        Item it = lista.getSelectedValue();
        if (it == null) {
            lblInfo.setText(" ");
            return;
        }
        if (it.tipo == Tipo.IMAGEM) {
            double[] n = tamanhoNatural(it, 0);
            lblInfo.setText(String.format("Imagem %d x %d px | DPI: %d%s | tamanho real: %.1f x %.1f cm",
                    it.img.getWidth(), it.img.getHeight(), dpiEfetivo(it),
                    it.dpiArquivo > 0 ? " (do arquivo)" : "", n[0] / 72 * 2.54, n[1] / 72 * 2.54));
        } else if (it.tipo == Tipo.PDF) {
            lblInfo.setText("PDF com " + it.paginasPdf + " página(s)");
        } else {
            lblInfo.setText("Texto com " + it.linhas.size() + " linhas");
        }
    }

    // =====================================================================
    //  Adicionar arquivos
    // =====================================================================

    private void escolherArquivos() {
        JFileChooser fc = new JFileChooser(ultimaPasta);
        fc.setMultiSelectionEnabled(true);
        fc.setFileFilter(new FileNameExtensionFilter("Imagens, PDF e textos",
                "jpg", "jpeg", "png", "gif", "bmp", "tif", "tiff", "webp", "pdf", "txt", "log", "java", "csv", "md",
                "xml", "json", "html"));
        if (fc.showOpenDialog(this) != JFileChooser.APPROVE_OPTION) return;
        ultimaPasta = fc.getCurrentDirectory();
        adicionar(Arrays.asList(fc.getSelectedFiles()));
    }

    private void escolherPasta() {
        JFileChooser fc = new JFileChooser(ultimaPasta);
        fc.setFileSelectionMode(JFileChooser.DIRECTORIES_ONLY);
        if (fc.showOpenDialog(this) != JFileChooser.APPROVE_OPTION) return;
        ultimaPasta = fc.getSelectedFile();
        adicionar(Arrays.asList(fc.getSelectedFile()));
    }

    private void adicionar(List<File> arquivos) {
        List<File> todos = new ArrayList<>();
        for (File f : arquivos) {
            if (f.isDirectory()) {
                File[] fs = f.listFiles(File::isFile);
                if (fs != null) {
                    Arrays.sort(fs, (a, b) -> a.getName().compareToIgnoreCase(b.getName()));
                    todos.addAll(Arrays.asList(fs));
                }
            } else {
                todos.add(f);
            }
        }
        lblStatus.setText("Carregando " + todos.size() + " arquivo(s)...");
        new SwingWorker<Void, Item>() {
            int ignorados = 0;
            String ultimoErro = "";

            @Override
            protected Void doInBackground() {
                for (File f : todos) {
                    try {
                        publish(carregar(f));
                    } catch (Exception e) {
                        ignorados++;
                        ultimoErro = e.getMessage();
                    }
                }
                return null;
            }

            @Override
            protected void process(List<Item> pedacos) {
                for (Item it : pedacos) modelo.addElement(it);
                if (lista.getSelectedIndex() < 0 && !modelo.isEmpty()) lista.setSelectedIndex(0);
            }

            @Override
            protected void done() {
                lblStatus.setText(modelo.size() + " arquivo(s) na fila"
                        + (ignorados > 0 ? " | " + ignorados + " ignorado(s): " + ultimoErro : ""));
                previa.repaint();
            }
        }.execute();
    }

    private Item carregar(File f) throws Exception {
        String n = f.getName().toLowerCase();
        String ext = n.contains(".") ? n.substring(n.lastIndexOf('.') + 1) : "";
        Item it = new Item();
        it.arquivo = f;
        if (ext.equals("pdf")) {
            carregarPdf(it);
            it.tipo = Tipo.PDF;
            try {
                it.miniPdf = renderPdf(it, 0, 50f);
            } catch (Exception ignorada) {
                // sem miniatura
            }
            return it;
        }
        if (TEXTOS.contains(ext)) {
            try {
                it.linhas = Files.readAllLines(f.toPath(), StandardCharsets.UTF_8);
            } catch (java.nio.charset.MalformedInputException e) {
                it.linhas = Files.readAllLines(f.toPath(), StandardCharsets.ISO_8859_1);
            }
            it.tipo = Tipo.TEXTO;
            return it;
        }
        BufferedImage img = lerImagem(f);
        if (img == null) throw new IOException("formato não suportado (" + f.getName() + ")");
        it.tipo = Tipo.IMAGEM;
        it.img = img;
        it.dpiArquivo = dpiDoArquivo(f);
        double fator = Math.min(1.0, 700.0 / Math.max(img.getWidth(), img.getHeight()));
        it.mini = redimensionar(img, Math.max(1, (int) (img.getWidth() * fator)),
                Math.max(1, (int) (img.getHeight() * fator)));
        return it;
    }

    // =====================================================================
    //  Leitura de imagens complexas
    // =====================================================================

    private BufferedImage lerImagem(File f) {
        BufferedImage img = null;
        try {
            img = ImageIO.read(f);
        } catch (Exception ignorada) {
            // JPEG CMYK/YCCK costuma cair aqui
        }
        if (img == null) img = lerJpegCmyk(f);
        if (img == null) {
            ImageIcon ic = new ImageIcon(f.getPath());
            if (ic.getIconWidth() > 0 && ic.getIconHeight() > 0) {
                img = new BufferedImage(ic.getIconWidth(), ic.getIconHeight(), BufferedImage.TYPE_INT_ARGB);
                Graphics2D g = img.createGraphics();
                g.drawImage(ic.getImage(), 0, 0, null);
                g.dispose();
            }
        }
        if (img == null) return null;
        img = paraRgbComFundoBranco(img);
        int o = orientacaoExif(f);
        return o > 1 ? girar(img, o) : img;
    }

    private BufferedImage paraRgbComFundoBranco(BufferedImage src) {
        BufferedImage rgb = new BufferedImage(src.getWidth(), src.getHeight(), BufferedImage.TYPE_INT_RGB);
        Graphics2D g = rgb.createGraphics();
        g.setColor(Color.WHITE);
        g.fillRect(0, 0, rgb.getWidth(), rgb.getHeight());
        g.drawImage(src, 0, 0, null);
        g.dispose();
        return rgb;
    }

    private BufferedImage lerJpegCmyk(File f) {
        try (ImageInputStream in = ImageIO.createImageInputStream(f)) {
            Iterator<ImageReader> it = ImageIO.getImageReaders(in);
            if (!it.hasNext()) return null;
            ImageReader reader = it.next();
            reader.setInput(in);
            Raster r = reader.readRaster(0, null);
            if (r.getNumBands() != 4) return null;
            int w = r.getWidth(), h = r.getHeight();
            BufferedImage out = new BufferedImage(w, h, BufferedImage.TYPE_INT_RGB);
            int[] p = new int[4];
            for (int y = 0; y < h; y++) {
                for (int x = 0; x < w; x++) {
                    r.getPixel(x, y, p);
                    double Y = p[0], cb = p[1] - 128, cr = p[2] - 128, k = p[3] / 255.0;
                    int R = (int) (limitar(Y + 1.402 * cr) * k);
                    int G = (int) (limitar(Y - 0.344136 * cb - 0.714136 * cr) * k);
                    int B = (int) (limitar(Y + 1.772 * cb) * k);
                    out.setRGB(x, y, (R << 16) | (G << 8) | B);
                }
            }
            return out;
        } catch (Exception e) {
            return null;
        }
    }

    private double limitar(double v) {
        return Math.max(0, Math.min(255, v));
    }

    private int dpiDoArquivo(File f) {
        try (ImageInputStream in = ImageIO.createImageInputStream(f)) {
            Iterator<ImageReader> it = ImageIO.getImageReaders(in);
            if (!it.hasNext()) return 0;
            ImageReader reader = it.next();
            reader.setInput(in);
            IIOMetadata md = reader.getImageMetadata(0);
            IIOMetadataNode raiz = (IIOMetadataNode) md.getAsTree("javax_imageio_1.0");
            NodeList nl = raiz.getElementsByTagName("HorizontalPixelSize");
            if (nl.getLength() == 0) return 0;
            double mm = Double.parseDouble(((IIOMetadataNode) nl.item(0)).getAttribute("value"));
            int dpi = (int) Math.round(25.4 / mm);
            return (dpi >= 36 && dpi <= 1200) ? dpi : 0;
        } catch (Exception e) {
            return 0;
        }
    }

    private int orientacaoExif(File f) {
        try (java.io.RandomAccessFile raf = new java.io.RandomAccessFile(f, "r")) {
            if (raf.readUnsignedShort() != 0xFFD8) return 1;
            while (true) {
                int marcador = raf.readUnsignedShort();
                int tam = raf.readUnsignedShort();
                if (marcador == 0xFFE1) {
                    byte[] d = new byte[tam - 2];
                    raf.readFully(d);
                    if (d.length > 14 && d[0] == 'E' && d[1] == 'x' && d[2] == 'i' && d[3] == 'f') {
                        int t = 6;
                        boolean le = d[t] == 'I';
                        int ifd = t + u32(d, t + 4, le);
                        int n = u16(d, ifd, le);
                        for (int i = 0; i < n; i++) {
                            int e = ifd + 2 + i * 12;
                            if (u16(d, e, le) == 0x0112) {
                                int o = u16(d, e + 8, le);
                                return (o >= 1 && o <= 8) ? o : 1;
                            }
                        }
                        return 1;
                    }
                } else if ((marcador & 0xFF00) != 0xFF00 || marcador == 0xFFDA) {
                    return 1;
                } else {
                    raf.skipBytes(tam - 2);
                }
            }
        } catch (Exception e) {
            return 1;
        }
    }

    private int u16(byte[] d, int o, boolean le) {
        int a = d[o] & 0xFF, b = d[o + 1] & 0xFF;
        return le ? (b << 8) | a : (a << 8) | b;
    }

    private int u32(byte[] d, int o, boolean le) {
        return le ? (u16(d, o + 2, true) << 16) | u16(d, o, true)
                : (u16(d, o, false) << 16) | u16(d, o + 2, false);
    }

    private BufferedImage girar(BufferedImage img, int o) {
        int w = img.getWidth(), h = img.getHeight();
        AffineTransform t = new AffineTransform();
        switch (o) {
            case 2: t.translate(w, 0); t.scale(-1, 1); break;
            case 3: t.translate(w, h); t.rotate(Math.PI); break;
            case 4: t.scale(1, -1); t.translate(0, -h); break;
            case 5: t.rotate(Math.PI / 2); t.scale(1, -1); break;
            case 6: t.translate(h, 0); t.rotate(Math.PI / 2); break;
            case 7: t.translate(h, w); t.rotate(3 * Math.PI / 2); t.scale(1, -1); break;
            case 8: t.translate(0, w); t.rotate(-Math.PI / 2); break;
            default: return img;
        }
        boolean troca = o >= 5;
        BufferedImage out = new BufferedImage(troca ? h : w, troca ? w : h, BufferedImage.TYPE_INT_RGB);
        Graphics2D g = out.createGraphics();
        g.setRenderingHint(RenderingHints.KEY_INTERPOLATION, RenderingHints.VALUE_INTERPOLATION_BICUBIC);
        g.drawImage(img, t, null);
        g.dispose();
        return out;
    }

    // =====================================================================
    //  PDF (PDFBox opcional, via reflexão: compila mesmo sem a biblioteca)
    // =====================================================================

    private void carregarPdf(Item it) throws Exception {
        Object doc;
        try {
            Class<?> loader = Class.forName("org.apache.pdfbox.Loader"); // PDFBox 3.x
            doc = loader.getMethod("loadPDF", File.class).invoke(null, it.arquivo);
        } catch (ClassNotFoundException e) {
            try {
                Class<?> pd = Class.forName("org.apache.pdfbox.pdmodel.PDDocument"); // PDFBox 2.x
                doc = pd.getMethod("load", File.class).invoke(null, it.arquivo);
            } catch (ClassNotFoundException e2) {
                throw new IOException("PDF precisa do PDFBox (veja as instruções no topo do código)");
            }
        }
        it.pdfDoc = doc;
        it.paginasPdf = ((Number) doc.getClass().getMethod("getNumberOfPages").invoke(doc)).intValue();
    }

    private BufferedImage renderPdf(Item it, int pagina, float dpi) throws Exception {
        synchronized (it) {
            Class<?> pd = Class.forName("org.apache.pdfbox.pdmodel.PDDocument");
            Object r = Class.forName("org.apache.pdfbox.rendering.PDFRenderer")
                    .getConstructor(pd).newInstance(it.pdfDoc);
            return (BufferedImage) r.getClass().getMethod("renderImageWithDPI", int.class, float.class)
                    .invoke(r, pagina, dpi);
        }
    }

    private double[] tamanhoPdf(Item it, int p) {
        if (it.tamPdf == null) it.tamPdf = new double[it.paginasPdf][];
        if (it.tamPdf[p] == null) {
            double w = 595.28, h = 841.89;
            try {
                synchronized (it) {
                    Object page = it.pdfDoc.getClass().getMethod("getPage", int.class).invoke(it.pdfDoc, p);
                    Object box = page.getClass().getMethod("getMediaBox").invoke(page);
                    w = ((Number) box.getClass().getMethod("getWidth").invoke(box)).doubleValue();
                    h = ((Number) box.getClass().getMethod("getHeight").invoke(box)).doubleValue();
                    int rot = ((Number) page.getClass().getMethod("getRotation").invoke(page)).intValue();
                    if (rot == 90 || rot == 270) {
                        double t = w;
                        w = h;
                        h = t;
                    }
                }
            } catch (Exception ignorada) {
                // usa A4
            }
            it.tamPdf[p] = new double[]{w, h};
        }
        return it.tamPdf[p];
    }

    // =====================================================================
    //  Layout da página e resolução
    // =====================================================================

    private int dpiEfetivo(Item it) {
        return (chkDpiArq.isSelected() && it.dpiArquivo > 0) ? it.dpiArquivo : ((Number) spnDpi.getValue()).intValue();
    }

    /** Tamanho natural em pontos (1 pt = 1/72 pol.). */
    private double[] tamanhoNatural(Item it, int pagina) {
        if (it.tipo == Tipo.IMAGEM) {
            double dpi = dpiEfetivo(it);
            return new double[]{it.img.getWidth() * 72.0 / dpi, it.img.getHeight() * 72.0 / dpi};
        }
        if (it.tipo == Tipo.PDF) return tamanhoPdf(it, pagina);
        return new double[]{595.28, 841.89};
    }

    private PageFormat montarPageFormat(Item it) {
        int pi = cmbPapel.getSelectedIndex();
        double w = PAPEL_PT[pi][0], h = PAPEL_PT[pi][1];
        double m = ((Number) spnMargem.getValue()).doubleValue() * 72 / 25.4;
        Paper p = new Paper();
        p.setSize(w, h);
        p.setImageableArea(m, m, w - 2 * m, h - 2 * m);
        PageFormat pf = new PageFormat();
        pf.setPaper(p);
        int o = cmbOrient.getSelectedIndex();
        boolean paisagem = o == 2;
        if (o == 0 && it != null && it.tipo != Tipo.TEXTO) {
            double[] n = tamanhoNatural(it, 0);
            paisagem = n[0] > n[1];
        }
        pf.setOrientation(paisagem ? PageFormat.LANDSCAPE : PageFormat.PORTRAIT);
        return pf;
    }

    private Colocacao colocar(double nw, double nh, PageFormat pf, boolean permitirLadrilho) {
        double aw = pf.getImageableWidth(), ah = pf.getImageableHeight();
        double s = 1;
        if (rbAjustar.isSelected()) s = Math.min(aw / nw, ah / nh);
        else if (rbPreencher.isSelected()) s = Math.max(aw / nw, ah / nh);
        else if (rbPers.isSelected()) s = ((Number) spnPct.getValue()).doubleValue() / 100.0;
        Colocacao c = new Colocacao();
        c.w = nw * s;
        c.h = nh * s;
        boolean ladrilho = permitirLadrilho && chkDividir.isSelected() && (rb100.isSelected() || rbPers.isSelected());
        if (ladrilho) {
            c.colunas = Math.max(1, (int) Math.ceil(c.w / aw - 1e-6));
            c.linhas = Math.max(1, (int) Math.ceil(c.h / ah - 1e-6));
        }
        c.x = (chkCentro.isSelected() && c.colunas == 1) ? (aw - c.w) / 2 : 0;
        c.y = (chkCentro.isSelected() && c.linhas == 1) ? (ah - c.h) / 2 : 0;
        return c;
    }

    private int dpiSeguro(double wPt, double hPt) {
        int dpi = DPIS[cmbQualidade.getSelectedIndex()];
        while (dpi > 100 && (wPt / 72 * dpi) * (hPt / 72 * dpi) > 50_000_000d) dpi -= 50;
        return dpi;
    }

    private BufferedImage rasterImagem(Item it, double wPt, double hPt) {
        int dpi = dpiSeguro(wPt, hPt);
        int tw = Math.max(1, (int) Math.round(wPt / 72 * dpi));
        int th = Math.max(1, (int) Math.round(hPt / 72 * dpi));
        boolean nit = chkNitidez.isSelected();
        String chave = "I:" + tw + "x" + th + ":" + nit;
        synchronized (it) {
            if (chave.equals(it.cacheChave) && it.cacheImg != null) return it.cacheImg;
        }
        BufferedImage r = redimensionar(it.img, tw, th);
        if (nit) r = nitidez(r, 0.6f);
        synchronized (it) {
            it.cacheChave = chave;
            it.cacheImg = r;
        }
        return r;
    }

    private BufferedImage rasterPdf(Item it, int pagina, double nw, double wPt, double hPt) throws Exception {
        int dpi = dpiSeguro(wPt, hPt);
        float renderDpi = (float) Math.max(36, dpi * (wPt / nw));
        String chave = "P:" + pagina + ":" + Math.round(renderDpi);
        synchronized (it) {
            if (chave.equals(it.cacheChave) && it.cacheImg != null) return it.cacheImg;
        }
        BufferedImage r = renderPdf(it, pagina, renderDpi);
        synchronized (it) {
            it.cacheChave = chave;
            it.cacheImg = r;
        }
        return r;
    }

    private BufferedImage redimensionar(BufferedImage src, int tw, int th) {
        BufferedImage atual = src;
        int w = src.getWidth(), h = src.getHeight();
        while (w != tw || h != th) {
            w = passo(w, tw);
            h = passo(h, th);
            BufferedImage tmp = new BufferedImage(w, h, BufferedImage.TYPE_INT_RGB);
            Graphics2D g = tmp.createGraphics();
            g.setRenderingHint(RenderingHints.KEY_INTERPOLATION, RenderingHints.VALUE_INTERPOLATION_BICUBIC);
            g.setRenderingHint(RenderingHints.KEY_RENDERING, RenderingHints.VALUE_RENDER_QUALITY);
            g.drawImage(atual, 0, 0, w, h, null);
            g.dispose();
            atual = tmp;
        }
        return atual;
    }

    private int passo(int atual, int alvo) {
        if (alvo < atual) return Math.max(alvo, atual / 2);
        return Math.min(alvo, (int) Math.ceil(atual * 1.5));
    }

    private BufferedImage nitidez(BufferedImage src, float quantidade) {
        float[] k = {1 / 16f, 2 / 16f, 1 / 16f, 2 / 16f, 4 / 16f, 2 / 16f, 1 / 16f, 2 / 16f, 1 / 16f};
        BufferedImage borrada = new ConvolveOp(new Kernel(3, 3, k), ConvolveOp.EDGE_NO_OP, null).filter(src, null);
        int w = src.getWidth(), h = src.getHeight();
        BufferedImage out = new BufferedImage(w, h, BufferedImage.TYPE_INT_RGB);
        int[] a = new int[w], b = new int[w], c = new int[w];
        for (int y = 0; y < h; y++) {
            src.getRGB(0, y, w, 1, a, 0, w);
            borrada.getRGB(0, y, w, 1, b, 0, w);
            for (int x = 0; x < w; x++) {
                int rgb = 0;
                for (int s = 16; s >= 0; s -= 8) {
                    int o = (a[x] >> s) & 0xFF, bl = (b[x] >> s) & 0xFF;
                    int v = Math.round(o + quantidade * (o - bl));
                    rgb |= Math.max(0, Math.min(255, v)) << s;
                }
                c[x] = rgb;
            }
            out.setRGB(0, y, w, 1, c, 0, w);
        }
        return out;
    }

    // =====================================================================
    //  Impressão
    // =====================================================================

    private PrintRequestAttributeSet montarAtributos(PageFormat pf) {
        PrintRequestAttributeSet a = attrsDialogo != null
                ? new HashPrintRequestAttributeSet(attrsDialogo) : new HashPrintRequestAttributeSet();
        if (attrsDialogo == null) {
            a.add(new Copies(((Number) spnCopias.getValue()).intValue()));
            a.add(cmbCor.getSelectedIndex() == 0 ? Chromaticity.COLOR : Chromaticity.MONOCHROME);
            a.add(new PrintQuality[]{PrintQuality.DRAFT, PrintQuality.NORMAL, PrintQuality.HIGH, PrintQuality.HIGH}
                    [cmbQualidade.getSelectedIndex()]);
            a.add(new Sides[]{Sides.ONE_SIDED, Sides.DUPLEX, Sides.TUMBLE}[cmbLados.getSelectedIndex()]);
        }
        a.add(PAPEL_MEDIA[cmbPapel.getSelectedIndex()]);
        a.add(pf.getOrientation() == PageFormat.LANDSCAPE ? OrientationRequested.LANDSCAPE
                : OrientationRequested.PORTRAIT);
        return a;
    }

    private void imprimirFila() {
        if (modelo.isEmpty()) {
            JOptionPane.showMessageDialog(this, "A fila está vazia. Adicione arquivos primeiro.");
            return;
        }
        PrintService svc = (PrintService) cmbImpressora.getSelectedItem();
        attrsDialogo = null;
        if (chkDialogo.isSelected()) {
            try {
                PrinterJob j = PrinterJob.getPrinterJob();
                if (svc != null) j.setPrintService(svc);
                PrintRequestAttributeSet a = new HashPrintRequestAttributeSet();
                if (!j.printDialog(a)) return;
                svc = j.getPrintService();
                attrsDialogo = a;
            } catch (PrinterException e) {
                JOptionPane.showMessageDialog(this, "Erro na impressora: " + e.getMessage());
                return;
            }
        }
        final PrintService servico = svc;
        List<Item> itens = new ArrayList<>();
        for (int i = 0; i < modelo.size(); i++) {
            Item it = modelo.get(i);
            it.status = "Na fila";
            itens.add(it);
        }
        lista.repaint();
        cancelar = false;
        btnImprimir.setEnabled(false);
        btnCancelar.setEnabled(true);
        barra.setValue(0);

        SwingWorker<Void, Void> worker = new SwingWorker<Void, Void>() {
            @Override
            protected Void doInBackground() {
                for (int i = 0; i < itens.size(); i++) {
                    Item it = itens.get(i);
                    if (cancelar) {
                        it.status = "Cancelado";
                        continue;
                    }
                    it.status = "Imprimindo...";
                    SwingUtilities.invokeLater(lista::repaint);
                    try {
                        PageFormat pf = montarPageFormat(it);
                        Printable pr = it.tipo == Tipo.TEXTO ? new TextoPrintable(it) : new GraficoPrintable(it);
                        PrinterJob job = PrinterJob.getPrinterJob();
                        if (servico != null) job.setPrintService(servico);
                        job.setJobName("ImpressoraPro - " + it.arquivo.getName());
                        job.setPrintable(pr, pf);
                        job.print(montarAtributos(pf));
                        it.status = "Impresso ✓";
                    } catch (Exception e) {
                        it.status = "Erro: " + e.getMessage();
                    }
                    setProgress((i + 1) * 100 / itens.size());
                    SwingUtilities.invokeLater(lista::repaint);
                }
                return null;
            }

            @Override
            protected void done() {
                btnImprimir.setEnabled(true);
                btnCancelar.setEnabled(false);
                lblStatus.setText(cancelar ? "Impressão cancelada." : "Fila concluída. Confira o status de cada arquivo.");
                lista.repaint();
            }
        };
        worker.addPropertyChangeListener(e -> {
            if ("progress".equals(e.getPropertyName())) barra.setValue((Integer) e.getNewValue());
        });
        worker.execute();
    }

    /** Imagens e PDF: coloca na folha conforme escala, centralização e divisão em folhas. */
    private class GraficoPrintable implements Printable {
        private final Item it;

        GraficoPrintable(Item it) {
            this.it = it;
        }

        @Override
        public int print(Graphics g, PageFormat pf, int idx) throws PrinterException {
            double aw = pf.getImageableWidth(), ah = pf.getImageableHeight();
            double[] nat;
            Colocacao c;
            int pagina = 0, col = 0, lin = 0;
            if (it.tipo == Tipo.PDF) {
                if (idx >= it.paginasPdf) return NO_SUCH_PAGE;
                pagina = idx;
                nat = tamanhoNatural(it, idx);
                c = colocar(nat[0], nat[1], pf, false);
            } else {
                nat = tamanhoNatural(it, 0);
                c = colocar(nat[0], nat[1], pf, true);
                if (idx >= c.colunas * c.linhas) return NO_SUCH_PAGE;
                col = idx % c.colunas;
                lin = idx / c.colunas;
            }
            BufferedImage raster;
            try {
                raster = it.tipo == Tipo.PDF ? rasterPdf(it, pagina, nat[0], c.w, c.h) : rasterImagem(it, c.w, c.h);
            } catch (Exception e) {
                throw new PrinterException(String.valueOf(e.getMessage()));
            }
            Graphics2D g2 = (Graphics2D) g.create();
            g2.setRenderingHint(RenderingHints.KEY_INTERPOLATION, RenderingHints.VALUE_INTERPOLATION_BICUBIC);
            g2.setRenderingHint(RenderingHints.KEY_RENDERING, RenderingHints.VALUE_RENDER_QUALITY);
            g2.setRenderingHint(RenderingHints.KEY_COLOR_RENDERING, RenderingHints.VALUE_COLOR_RENDER_QUALITY);
            g2.translate(pf.getImageableX(), pf.getImageableY());
            g2.setClip(0, 0, (int) Math.ceil(aw), (int) Math.ceil(ah));
            g2.translate(c.x - col * aw, c.y - lin * ah);
            g2.drawImage(raster, 0, 0, (int) Math.round(c.w), (int) Math.round(c.h), null);
            g2.dispose();
            return PAGE_EXISTS;
        }
    }

    /** Texto em fonte monoespaçada, com quebra de linha e paginação. */
    private class TextoPrintable implements Printable {
        private final Item it;

        TextoPrintable(Item it) {
            this.it = it;
        }

        @Override
        public int print(Graphics g, PageFormat pf, int idx) {
            float tam = ((Number) spnFonte.getValue()).floatValue();
            Graphics2D g2 = (Graphics2D) g.create();
            g2.translate(pf.getImageableX(), pf.getImageableY());
            g2.setFont(new Font(Font.MONOSPACED, Font.PLAIN, 10).deriveFont(tam));
            g2.setColor(Color.BLACK);
            FontMetrics fm = g2.getFontMetrics();
            int alturaLinha = fm.getHeight();
            int colunas = Math.max(1, (int) (pf.getImageableWidth() / Math.max(1, fm.charWidth('M'))));
            int porPagina = Math.max(1, (int) (pf.getImageableHeight() / alturaLinha));

            List<String> q = new ArrayList<>();
            for (String l : it.linhas) {
                l = l.replace("\t", "    ");
                if (l.isEmpty()) q.add("");
                for (int i = 0; i < l.length(); i += colunas) q.add(l.substring(i, Math.min(l.length(), i + colunas)));
            }
            int ini = idx * porPagina;
            if (ini >= q.size()) {
                g2.dispose();
                return NO_SUCH_PAGE;
            }
            int y = fm.getAscent();
            for (int i = ini; i < Math.min(q.size(), ini + porPagina); i++) {
                g2.drawString(q.get(i), 0, y);
                y += alturaLinha;
            }
            g2.dispose();
            return PAGE_EXISTS;
        }
    }

    // =====================================================================
    //  Prévia da folha (mostra exatamente onde o arquivo vai sair no papel)
    // =====================================================================

    private class Previa extends JPanel {
        Previa() {
            setBackground(new Color(80, 84, 92));
        }

        @Override
        protected void paintComponent(Graphics g0) {
            super.paintComponent(g0);
            Graphics2D g = (Graphics2D) g0.create();
            g.setRenderingHint(RenderingHints.KEY_ANTIALIASING, RenderingHints.VALUE_ANTIALIAS_ON);
            g.setRenderingHint(RenderingHints.KEY_INTERPOLATION, RenderingHints.VALUE_INTERPOLATION_BILINEAR);
            Item it = lista.getSelectedValue();
            int W = getWidth(), H = getHeight();
            g.setColor(Color.WHITE);
            if (it == null) {
                g.drawString("Adicione arquivos e selecione um para ver a prévia da folha.", 20, 30);
                g.dispose();
                return;
            }
            PageFormat pf = montarPageFormat(it);
            double pw = pf.getWidth(), ph = pf.getHeight();
            double esc = Math.max(0.05, Math.min((W - 40) / pw, (H - 60) / ph));
            int px = (int) ((W - pw * esc) / 2), py = (int) ((H - ph * esc) / 2) + 8;
            int pwI = (int) (pw * esc), phI = (int) (ph * esc);
            g.setColor(new Color(0, 0, 0, 80));
            g.fillRect(px + 5, py + 5, pwI, phI);
            g.setColor(Color.WHITE);
            g.fillRect(px, py, pwI, phI);

            double aw = pf.getImageableWidth(), ah = pf.getImageableHeight();
            int ix = px + (int) (pf.getImageableX() * esc), iy = py + (int) (pf.getImageableY() * esc);
            int iw = (int) (aw * esc), ih = (int) (ah * esc);
            g.setColor(new Color(110, 150, 255));
            g.setStroke(new BasicStroke(1f, BasicStroke.CAP_BUTT, BasicStroke.JOIN_MITER, 10f,
                    new float[]{4f, 4f}, 0f));
            g.drawRect(ix, iy, iw, ih);

            Graphics2D gc = (Graphics2D) g.create();
            gc.setClip(ix, iy, iw + 1, ih + 1);
            String legenda = "Folha " + PAPEIS[cmbPapel.getSelectedIndex()]
                    + (pf.getOrientation() == PageFormat.LANDSCAPE ? " (paisagem)" : " (retrato)");
            if (it.tipo == Tipo.TEXTO) {
                int tam = Math.max(4, (int) (((Number) spnFonte.getValue()).floatValue() * esc));
                gc.setFont(new Font(Font.MONOSPACED, Font.PLAIN, tam));
                gc.setColor(Color.BLACK);
                int y = iy + gc.getFontMetrics().getAscent();
                for (int i = 0; i < it.linhas.size() && y < iy + ih; i++) {
                    gc.drawString(it.linhas.get(i), ix, y);
                    y += gc.getFontMetrics().getHeight();
                }
            } else {
                BufferedImage mini = it.tipo == Tipo.IMAGEM ? it.mini : it.miniPdf;
                double[] nat = tamanhoNatural(it, 0);
                Colocacao c = colocar(nat[0], nat[1], pf, it.tipo == Tipo.IMAGEM);
                if (mini != null) {
                    gc.drawImage(mini, ix + (int) (c.x * esc), iy + (int) (c.y * esc),
                            (int) (c.w * esc), (int) (c.h * esc), null);
                }
                int total = it.tipo == Tipo.PDF ? it.paginasPdf : c.colunas * c.linhas;
                if (c.colunas * c.linhas > 1 && it.tipo == Tipo.IMAGEM) {
                    gc.setColor(new Color(230, 60, 60));
                    for (int k = 1; k < c.colunas; k++) {
                        int x = ix + (int) ((c.x + k * aw) * esc);
                        gc.drawLine(x, iy, x, iy + ih);
                    }
                    for (int k = 1; k < c.linhas; k++) {
                        int y = iy + (int) ((c.y + k * ah) * esc);
                        gc.drawLine(ix, y, ix + iw, y);
                    }
                }
                legenda += " | " + total + " folha(s) no total";
            }
            gc.dispose();
            g.setColor(Color.WHITE);
            g.setStroke(new BasicStroke(1f));
            g.drawString(legenda, px, py - 6);
            g.dispose();
        }
    }

    public static void main(String[] args) {
        try {
            UIManager.setLookAndFeel(UIManager.getSystemLookAndFeelClassName());
        } catch (Exception ignorada) {
            // usa o visual padrão
        }
        SwingUtilities.invokeLater(() -> new ImpressoraPro().setVisible(true));
    }
}
