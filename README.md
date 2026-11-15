# Flappy Bird — kodi Java

Ky është kodi Java i lojës i krijuar në Eclipse. Është ruajtur këtu për ta parë si tekst; nuk ekzekutohet nga versioni HTML.

```java
package game;

import java.awt.BasicStroke;
import java.awt.Color;
import java.awt.Dimension;
import java.awt.Font;
import java.awt.FontMetrics;
import java.awt.Graphics;
import java.awt.Graphics2D;
import java.awt.Polygon;
import java.awt.Rectangle;
import java.awt.RenderingHints;
import java.awt.event.KeyAdapter;
import java.awt.event.KeyEvent;
import java.awt.event.MouseAdapter;
import java.awt.event.MouseEvent;
import java.awt.geom.AffineTransform;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.ArrayList;
import java.util.List;
import java.util.Random;

import javax.swing.JFrame;
import javax.swing.JPanel;
import javax.swing.SwingUtilities;

/**
 * Flappy Bird - klon në Java 2D (Swing).
 * Kontrolli: SPACE / ↑ / W ose klik me maus = fluturo.
 */
public class mygame {

    // ---------- Konstantet e lojës ----------
    static final int GAME_WIDTH = 400;
    static final int GAME_HEIGHT = 640;
    static final int GROUND_HEIGHT = 100;
    static final int GROUND_Y = GAME_HEIGHT - GROUND_HEIGHT;

    static final double GRAVITY = 0.45;
    static final double FLAP_SPEED = -7.5;
    static final double MAX_FALL_SPEED = 11;

    static final double PIPE_SPEED = 2.5;
    static final int PIPE_WIDTH = 70;
    static final int PIPE_GAP = 150;
    static final int PIPE_SPACING = 210;

    public static void main(String[] args) {
        SwingUtilities.invokeLater(() -> {
            JFrame window = new JFrame("Flappy Bird");
            window.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);

            GamePanel panel = new GamePanel();
            window.add(panel);
            window.pack();
            window.setResizable(false);
            window.setLocationRelativeTo(null);
            window.setVisible(true);

            panel.startGameThread();
        });
    }

    enum State {
        READY, PLAYING, DEAD
    }

    // =====================================================
    //  ZOGU
    // =====================================================
    static class Bird {
        static final int W = 34;
        static final int H = 24;

        double x, y, vy;

        Bird(double x, double y) {
            this.x = x;
            this.y = y;
        }

        void flap() {
            vy = FLAP_SPEED;
        }

        void update() {
            vy += GRAVITY;
            if (vy > MAX_FALL_SPEED) {
                vy = MAX_FALL_SPEED;
            }
            y += vy;
            if (y < 0) { // tavani
                y = 0;
                vy = 0;
            }
        }

        Rectangle getHitbox() {
            return new Rectangle((int) x + 4, (int) y + 4, W - 8, H - 8);
        }

        void draw(Graphics2D g, int frame, boolean flapping) {
            AffineTransform old = g.getTransform();
            g.translate(x + W / 2.0, y + H / 2.0);
            double angle = Math.max(-0.45, Math.min(1.4, vy * 0.12));
            g.rotate(angle);

            int left = -W / 2;
            int top = -H / 2;

            // trupi
            g.setColor(new Color(248, 210, 50));
            g.fillOval(left, top, W, H);
            g.setColor(new Color(83, 56, 71));
            g.setStroke(new BasicStroke(2));
            g.drawOval(left, top, W, H);

            // krahu (lëviz kur zogu fluturon)
            int wingShift = flapping ? new int[] { -3, 0, 3, 0 }[(frame / 4) % 4] : 0;
            g.setColor(new Color(255, 245, 200));
            g.fillOval(left + 3, top + 9 + wingShift, 16, 10);
            g.setColor(new Color(83, 56, 71));
            g.drawOval(left + 3, top + 9 + wingShift, 16, 10);

            // syri
            g.setColor(Color.WHITE);
            g.fillOval(left + 20, top + 2, 12, 12);
            g.setColor(new Color(83, 56, 71));
            g.drawOval(left + 20, top + 2, 12, 12);
            g.fillOval(left + 26, top + 6, 4, 4);

            // sqepi
            g.setColor(new Color(240, 100, 40));
            g.fillRoundRect(left + 22, top + 13, 14, 8, 6, 6);
            g.setColor(new Color(83, 56, 71));
            g.drawRoundRect(left + 22, top + 13, 14, 8, 6, 6);

            g.setTransform(old);
        }
    }

    // =====================================================
    //  TUBAT
    // =====================================================
    static class Pipe {
        double x;
        final int gapY; // pozicioni i fillimit të hapësirës
        boolean scored;

        Pipe(double x, int gapY) {
            this.x = x;
            this.gapY = gapY;
        }

        void update() {
            x -= PIPE_SPEED;
        }

        boolean isOffScreen() {
            return x + PIPE_WIDTH < 0;
        }

        Rectangle topRect() {
            return new Rectangle((int) x, 0, PIPE_WIDTH, gapY);
        }

        Rectangle bottomRect() {
            int top = gapY + PIPE_GAP;
            return new Rectangle((int) x, top, PIPE_WIDTH, GROUND_Y - top);
        }

        void draw(Graphics2D g) {
            drawPipe(g, topRect(), true);
            drawPipe(g, bottomRect(), false);
        }

        private void drawPipe(Graphics2D g, Rectangle r, boolean isTop) {
            Color body = new Color(115, 191, 46);
            Color dark = new Color(83, 56, 71);
            Color light = new Color(176, 233, 100);

            g.setColor(body);
            g.fillRect(r.x + 3, r.y, r.width - 6, r.height);
            g.setColor(light);
            g.fillRect(r.x + 7, r.y, 8, r.height);
            g.setColor(dark);
            g.setStroke(new BasicStroke(2));
            g.drawRect(r.x + 3, r.y, r.width - 6, r.height);

            int capH = 26;
            int capY = isTop ? r.y + r.height - capH : r.y;
            g.setColor(body);
            g.fillRect(r.x, capY, r.width, capH);
            g.setColor(light);
            g.fillRect(r.x + 4, capY, 8, capH);
            g.setColor(dark);
            g.drawRect(r.x, capY, r.width, capH);
        }
    }

    // =====================================================
    //  PANELI KRYESOR
    // =====================================================
    static class GamePanel extends JPanel implements Runnable {

        private static final long serialVersionUID = 1L;
        private static final Path SCORE_FILE = Paths.get("flappy_highscore.txt");

        private final Random random = new Random();
        private Thread gameThread;
        private volatile boolean running;

        Bird bird;
        final List<Pipe> pipes = new ArrayList<>();
        State state = State.READY;

        int score;
        int best;
        int frame;
        int deadTimer;
        double groundOffset;
        boolean keyHeld;

        GamePanel() {
            setPreferredSize(new Dimension(GAME_WIDTH, GAME_HEIGHT));
            setFocusable(true);
            setDoubleBuffered(true);
            best = loadBest();
            reset();

            addKeyListener(new KeyAdapter() {
                @Override
                public void keyPressed(KeyEvent e) {
                    int k = e.getKeyCode();
                    if ((k == KeyEvent.VK_SPACE || k == KeyEvent.VK_UP || k == KeyEvent.VK_W) && !keyHeld) {
                        keyHeld = true;
                        flap();
                    }
                }

                @Override
                public void keyReleased(KeyEvent e) {
                    keyHeld = false;
                }
            });
            addMouseListener(new MouseAdapter() {
                @Override
                public void mousePressed(MouseEvent e) {
                    requestFocusInWindow();
                    flap();
                }
            });
        }

        // ---------- Loop ----------
        void startGameThread() {
            running = true;
            gameThread = new Thread(this, "game-loop");
            gameThread.start();
            requestFocusInWindow();
        }

        @Override
        public void run() {
            final double nsPerUpdate = 1_000_000_000.0 / 60;
            long last = System.nanoTime();
            double acc = 0;
            while (running) {
                long now = System.nanoTime();
                acc += (now - last) / nsPerUpdate;
                last = now;
                while (acc >= 1) {
                    update();
                    acc--;
                }
                repaint();
                try {
                    Thread.sleep(2);
                } catch (InterruptedException e) {
                    running = false;
                }
            }
        }

        // ---------- Logjika ----------
        void reset() {
            bird = new Bird(100, 280);
            pipes.clear();
            score = 0;
            deadTimer = 0;
            state = State.READY;
        }

        void flap() {
            switch (state) {
                case READY:
                    state = State.PLAYING;
                    bird.flap();
                    break;
                case PLAYING:
                    bird.flap();
                    break;
                case DEAD:
                    if (deadTimer > 40) {
                        reset();
                    }
                    break;
                default:
                    break;
            }
        }

        void update() {
            frame++;
            switch (state) {
                case READY:
                    groundOffset += PIPE_SPEED;
                    bird.y = 280 + Math.sin(frame * 0.12) * 8;
                    break;
                case PLAYING:
                    updatePlaying();
                    break;
                case DEAD:
                    updateDead();
                    break;
                default:
                    break;
            }
        }

        private void updatePlaying() {
            groundOffset += PIPE_SPEED;
            bird.update();

            // krijo tuba të reja
            if (pipes.isEmpty() || pipes.get(pipes.size() - 1).x < GAME_WIDTH - PIPE_SPACING) {
                int minGap = 80;
                int maxGap = GROUND_Y - PIPE_GAP - 80;
                pipes.add(new Pipe(GAME_WIDTH, minGap + random.nextInt(maxGap - minGap + 1)));
            }

            Rectangle hit = bird.getHitbox();
            boolean crashed = bird.y + Bird.H >= GROUND_Y;

            for (int i = pipes.size() - 1; i >= 0; i--) {
                Pipe p = pipes.get(i);
                p.update();
                if (p.isOffScreen()) {
                    pipes.remove(i);
                    continue;
                }
                if (!p.scored && p.x + PIPE_WIDTH < bird.x) {
                    p.scored = true;
                    score++;
                }
                if (hit.intersects(p.topRect()) || hit.intersects(p.bottomRect())) {
                    crashed = true;
                }
            }

            if (crashed) {
                state = State.DEAD;
                deadTimer = 0;
                if (score > best) {
                    best = score;
                    saveBest(best);
                }
            }
        }

        private void updateDead() {
            deadTimer++;
            if (bird.y + Bird.H < GROUND_Y) {
                bird.vy += GRAVITY;
                if (bird.vy > MAX_FALL_SPEED) {
                    bird.vy = MAX_FALL_SPEED;
                }
                bird.y += bird.vy;
                if (bird.y + Bird.H > GROUND_Y) {
                    bird.y = GROUND_Y - Bird.H;
                }
            }
        }

        // ---------- Rezultati i ruajtur ----------
        private int loadBest() {
            try {
                if (Files.exists(SCORE_FILE)) {
                    return Integer.parseInt(Files.readAllLines(SCORE_FILE).get(0).trim());
                }
            } catch (IOException | RuntimeException e) {
                // fillojmë nga 0
            }
            return 0;
        }

        private void saveBest(int value) {
            try {
                Files.write(SCORE_FILE, String.valueOf(value).getBytes());
            } catch (IOException e) {
                System.err.println("Nuk u ruajt rekordi: " + e.getMessage());
            }
        }

        // ---------- Vizatimi ----------
        @Override
        protected void paintComponent(Graphics gr) {
            super.paintComponent(gr);
            Graphics2D g = (Graphics2D) gr;
            g.setRenderingHint(RenderingHints.KEY_ANTIALIASING, RenderingHints.VALUE_ANTIALIAS_ON);
            g.setRenderingHint(RenderingHints.KEY_TEXT_ANTIALIASING, RenderingHints.VALUE_TEXT_ANTIALIAS_ON);

            drawBackground(g);
            for (Pipe p : pipes) {
                p.draw(g);
            }
            drawGround(g);
            bird.draw(g, frame, state != State.DEAD);

            switch (state) {
                case READY:
                    drawReady(g);
                    break;
                case PLAYING:
                    drawScore(g);
                    break;
                case DEAD:
                    if (deadTimer < 8) { // vezullim i bardhë në përplasje
                        g.setColor(new Color(255, 255, 255, 200 - deadTimer * 25));
                        g.fillRect(0, 0, GAME_WIDTH, GAME_HEIGHT);
                    }
                    if (deadTimer > 20) {
                        drawGameOver(g);
                    }
                    break;
                default:
                    break;
            }
        }

        private void drawBackground(Graphics2D g) {
            g.setColor(new Color(112, 197, 206));
            g.fillRect(0, 0, GAME_WIDTH, GROUND_Y);

            // retë
            g.setColor(new Color(255, 255, 255, 190));
            for (int i = 0; i < 4; i++) {
                int cx = (int) (((i * 170 - frame * 0.3) % (GAME_WIDTH + 140) + GAME_WIDTH + 140) % (GAME_WIDTH + 140)) - 70;
                int cy = 360 + (i % 2) * 20;
                g.fillOval(cx, cy, 70, 30);
                g.fillOval(cx + 25, cy - 14, 50, 36);
                g.fillOval(cx + 45, cy, 60, 30);
            }

            // ndërtesat në sfond
            g.setColor(new Color(160, 222, 205));
            int[] heights = { 60, 90, 50, 110, 70, 95, 55, 80, 100, 65 };
            for (int i = 0; i < heights.length; i++) {
                g.fillRect(i * 42, GROUND_Y - heights[i], 36, heights[i]);
            }
            // shkurre
            g.setColor(new Color(140, 214, 108));
            for (int i = 0; i < 9; i++) {
                g.fillOval(i * 50 - 10, GROUND_Y - 22, 62, 40);
            }
        }

        private void drawGround(Graphics2D g) {
            g.setColor(new Color(222, 216, 149));
            g.fillRect(0, GROUND_Y, GAME_WIDTH, GROUND_HEIGHT);

            g.setColor(new Color(115, 191, 46));
            g.fillRect(0, GROUND_Y, GAME_WIDTH, 14);
            g.setColor(new Color(155, 225, 75));
            int shift = (int) (groundOffset % 24);
            for (int x = -24; x < GAME_WIDTH + 24; x += 24) {
                int sx = x - shift;
                g.fillPolygon(new Polygon(
                        new int[] { sx, sx + 12, sx + 24, sx + 12 },
                        new int[] { GROUND_Y, GROUND_Y, GROUND_Y + 14, GROUND_Y + 14 }, 4));
            }
            g.setColor(new Color(83, 56, 71));
            g.fillRect(0, GROUND_Y - 2, GAME_WIDTH, 3);
            g.setColor(new Color(213, 168, 80));
            g.fillRect(0, GROUND_Y + 14, GAME_WIDTH, 5);
        }

        private void drawReady(Graphics2D g) {
            g.setFont(new Font("SansSerif", Font.BOLD, 40));
            drawOutlined(g, "Get Ready!", centerX(g, "Get Ready!"), 150, new Color(252, 160, 72));

            g.setFont(new Font("SansSerif", Font.BOLD, 18));
            String hint = "Shtyp SPACE ose kliko p\u00ebr t\u00eb fluturuar";
            drawOutlined(g, hint, centerX(g, hint), 400, Color.WHITE);

            g.setFont(new Font("SansSerif", Font.BOLD, 16));
            String bestText = "Rekordi: " + best;
            drawOutlined(g, bestText, centerX(g, bestText), 440, Color.WHITE);
        }

        private void drawScore(Graphics2D g) {
            g.setFont(new Font("SansSerif", Font.BOLD, 56));
            String s = String.valueOf(score);
            drawOutlined(g, s, centerX(g, s), 100, Color.WHITE);
        }

        private void drawGameOver(Graphics2D g) {
            g.setFont(new Font("SansSerif", Font.BOLD, 42));
            drawOutlined(g, "Game Over", centerX(g, "Game Over"), 150, new Color(252, 160, 72));

            int pw = 280, ph = 150;
            int px = (GAME_WIDTH - pw) / 2, py = 190;
            g.setColor(new Color(222, 216, 149));
            g.fillRoundRect(px, py, pw, ph, 14, 14);
            g.setColor(new Color(83, 56, 71));
            g.setStroke(new BasicStroke(3));
            g.drawRoundRect(px, py, pw, ph, 14, 14);

            // medalja
            g.setFont(new Font("SansSerif", Font.BOLD, 14));
            g.setColor(new Color(220, 120, 40));
            g.drawString("MEDALJA", px + 20, py + 28);
            Color medal = medalColor();
            g.setColor(new Color(160, 150, 100));
            g.fillOval(px + 22, py + 42, 66, 66);
            if (medal != null) {
                g.setColor(medal);
                g.fillOval(px + 26, py + 46, 58, 58);
                g.setColor(new Color(255, 255, 255, 120));
                g.fillOval(px + 36, py + 54, 18, 18);
            }
            g.setColor(new Color(83, 56, 71));
            g.drawOval(px + 22, py + 42, 66, 66);

            // rezultatet
            g.setFont(new Font("SansSerif", Font.BOLD, 14));
            g.setColor(new Color(220, 120, 40));
            g.drawString("REZULTATI", px + 170, py + 28);
            g.drawString("REKORDI", px + 170, py + 88);
            g.setFont(new Font("SansSerif", Font.BOLD, 30));
            drawOutlined(g, String.valueOf(score), px + 170, py + 62, Color.WHITE);
            drawOutlined(g, String.valueOf(best), px + 170, py + 122, Color.WHITE);

            if (deadTimer > 40) {
                g.setFont(new Font("SansSerif", Font.BOLD, 18));
                String again = "Shtyp SPACE p\u00ebr t\u00eb rifilluar";
                drawOutlined(g, again, centerX(g, again), 385, Color.WHITE);
            }
        }

        private Color medalColor() {
            if (score >= 40) {
                return new Color(229, 228, 226); // platin
            }
            if (score >= 30) {
                return new Color(255, 200, 40);  // ar
            }
            if (score >= 20) {
                return new Color(192, 192, 200); // argjend
            }
            if (score >= 10) {
                return new Color(205, 127, 50);  // bronz
            }
            return null;
        }

        private int centerX(Graphics2D g, String text) {
            FontMetrics fm = g.getFontMetrics();
            return (GAME_WIDTH - fm.stringWidth(text)) / 2;
        }

        private void drawOutlined(Graphics2D g, String text, int x, int y, Color fill) {
            g.setColor(new Color(83, 56, 71));
            for (int dx = -2; dx <= 2; dx++) {
                for (int dy = -2; dy <= 2; dy++) {
                    g.drawString(text, x + dx, y + dy);
                }
            }
            g.setColor(fill);
            g.drawString(text, x, y);
        }
    }
}
```
