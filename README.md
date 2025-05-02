# log_analyser
this code is used to remove old logs

import java.io.*;
import java.nio.file.*;
import java.nio.file.attribute.BasicFileAttributes;
import java.nio.file.attribute.FileTime;
import java.time.*;
import java.util.zip.GZIPOutputStream;

public class LogArchiver {

    public static void main(String[] args) throws IOException {
        String sourceDir = "/var/log";
        String destDir = "./archived_logs";
        int daysOld = 1;
        boolean deleteOriginal = false;

        // Parse CLI args
        for (int i = 0; i < args.length; i++) {
            switch (args[i]) {
                case "--source":
                    sourceDir = args[++i];
                    break;
                case "--dest":
                    destDir = args[++i];
                    break;
                case "--days":
                    daysOld = Integer.parseInt(args[++i]);
                    break;
                case "--delete":
                    deleteOriginal = true;
                    break;
            }
        }

        // Final copies for lambda use
        final int finalDaysOld = daysOld;
        final String finalDestDir = destDir;
        final boolean finalDeleteOriginal = deleteOriginal;

        File dest = new File(finalDestDir);
        if (!dest.exists()) dest.mkdirs();

        Files.walk(Paths.get(sourceDir))
                .filter(Files::isRegularFile)
                .filter(p -> p.toString().endsWith(".log"))
                .forEach(path -> {
                    try {
                        BasicFileAttributes attrs = Files.readAttributes(path, BasicFileAttributes.class);
                        FileTime lastModifiedTime = attrs.lastModifiedTime();
                        Instant cutoff = Instant.now().minus(Duration.ofDays(finalDaysOld));

                        if (lastModifiedTime.toInstant().isBefore(cutoff)) {
                            File original = path.toFile();
                            String fileName = original.getName();
                            String dateStr = LocalDate.now().toString();
                            File archived = new File(finalDestDir, fileName + "." + dateStr + ".gz");

                            try (FileInputStream fis = new FileInputStream(original);
                                 FileOutputStream fos = new FileOutputStream(archived);
                                 GZIPOutputStream gos = new GZIPOutputStream(fos)) {
                                byte[] buffer = new byte[1024];
                                int len;
                                while ((len = fis.read(buffer)) != -1) {
                                    gos.write(buffer, 0, len);
                                }
                            }

                            if (finalDeleteOriginal) {
                                Files.delete(path);
                            }

                            System.out.println("Archived: " + path + " -> " + archived.getPath());
                        }
                    } catch (IOException e) {
                        System.err.println("Error archiving file: " + path);
                        e.printStackTrace();
                    }
                });
    }
}
