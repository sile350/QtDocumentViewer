# Руководство по разработке плагинов для Document Viewer

Это подробное руководство по созданию собственных плагинов для Document Viewer.

## 📋 Содержание

- [Введение](#введение)
- [Архитектура плагинов](#архитектура-плагинов)
- [Создание простого плагина](#создание-простого-плагина)
- [Интерфейс ViewerInterface](#интерфейс-viewerinterface)
- [Работа с UI](#работа-с-ui)
- [Сохранение состояния](#сохранение-состояния)
- [Печать документов](#печать-документов)
- [Примеры](#примеры)
- [Лучшие практики](#лучшие-практики)

## 🎯 Введение

Document Viewer использует систему плагинов Qt для поддержки различных типов документов. Каждый тип документа обрабатывается отдельным плагином, который реализует интерфейс `ViewerInterface`.

### Преимущества системы плагинов

- ✅ **Модульность**: Каждый формат файла - отдельный модуль
- ✅ **Расширяемость**: Легко добавить новые форматы
- ✅ **Независимость**: Плагины не зависят друг от друга
- ✅ **Динамическая загрузка**: Плагины загружаются автоматически

## 🏗️ Архитектура плагинов

### Иерархия классов

```
QObject
    └── ViewerInterface (чистый интерфейс)
            └── AbstractViewer (базовая реализация)
                    └── Ваш плагин (конкретная реализация)
```

### Компоненты плагина

1. **Класс плагина** - реализует `ViewerInterface`
2. **JSON метаданные** - описывают плагин
3. **CMakeLists.txt** - конфигурация сборки
4. **UI компоненты** - виджеты для отображения

## 🚀 Создание простого плагина

Создадим простой плагин для просмотра CSV файлов.

### Шаг 1: Структура директорий

```
plugins/
└── csvviewer/
    ├── CMakeLists.txt
    ├── csvviewer.h
    ├── csvviewer.cpp
    └── csvviewer.json
```

### Шаг 2: Заголовочный файл (csvviewer.h)

```cpp
#ifndef CSVVIEWER_H
#define CSVVIEWER_H

#include "viewerinterfaces.h"

QT_BEGIN_NAMESPACE
class QTableView;
class QStandardItemModel;
QT_END_NAMESPACE

class CsvViewer : public ViewerInterface
{
    Q_OBJECT
    // Указываем IID интерфейса и JSON файл с метаданными
    Q_PLUGIN_METADATA(IID "org.qt-project.Qt.Examples.DocumentViewer.ViewerInterface" 
                      FILE "csvviewer.json")
    Q_INTERFACES(ViewerInterface)

public:
    CsvViewer();
    ~CsvViewer() override;

    // Обязательные методы интерфейса
    void init(QFile *file, QWidget *parent, QMainWindow *mainWindow) override;
    QString viewerName() const override;
    QStringList supportedMimeTypes() const override;
    bool hasContent() const override;
    
    // Опциональные методы
    QByteArray saveState() const override;
    bool restoreState(QByteArray &state) override;
    bool supportsOverview() const override { return false; }

#ifdef QT_DOCUMENTVIEWER_PRINTSUPPORT
protected:
    void printDocument(QPrinter *printer) const override;
#endif

private slots:
    void setupCsvUi();

private:
    bool parseFile();
    
    QTableView *m_tableView = nullptr;
    QStandardItemModel *m_model = nullptr;
};

#endif // CSVVIEWER_H
```

### Шаг 3: Файл реализации (csvviewer.cpp)

```cpp
#include "csvviewer.h"

#include <QFile>
#include <QTableView>
#include <QStandardItemModel>
#include <QTextStream>
#include <QHeaderView>
#include <QMainWindow>

#ifdef QT_DOCUMENTVIEWER_PRINTSUPPORT
#include <QPrinter>
#include <QPainter>
#endif

using namespace Qt::StringLiterals;

CsvViewer::CsvViewer()
    : m_model(new QStandardItemModel(this))
{
}

CsvViewer::~CsvViewer()
{
    // Умные указатели и родительские объекты Qt автоматически очистят память
}

void CsvViewer::init(QFile *file, QWidget *parent, QMainWindow *mainWindow)
{
    // Сохраняем указатели
    ViewerInterface::init(file, parent, mainWindow);
    
    // Создаем UI
    m_tableView = new QTableView(parent);
    m_tableView->setModel(m_model);
    m_tableView->horizontalHeader()->setStretchLastSection(true);
    
    // Парсим файл
    if (parseFile()) {
        // Устанавливаем виджет для отображения
        statusString(tr("CSV file loaded successfully"));
    } else {
        statusString(tr("Failed to load CSV file"));
    }
    
    // Инициализируем UI после загрузки
    QMetaObject::invokeMethod(this, &CsvViewer::setupCsvUi, Qt::QueuedConnection);
}

QString CsvViewer::viewerName() const
{
    return QLatin1StringView(staticMetaObject.className());
}

QStringList CsvViewer::supportedMimeTypes() const
{
    // MIME-типы, которые поддерживает плагин
    return QStringList{
        "text/csv"_L1,
        "text/comma-separated-values"_L1,
        "application/csv"_L1
    };
}

bool CsvViewer::hasContent() const
{
    return m_model && m_model->rowCount() > 0;
}

void CsvViewer::setupCsvUi()
{
    // Уведомляем, что UI инициализирован
    uiInitialized(m_tableView);
    
    // Создаем меню и действия
    auto *viewMenu = menuBar()->addMenu(tr("&View"));
    
    auto *fitColumnsAction = viewMenu->addAction(tr("Fit Columns"));
    connect(fitColumnsAction, &QAction::triggered, this, [this]() {
        m_tableView->resizeColumnsToContents();
    });
    
    auto *resetAction = viewMenu->addAction(tr("Reset View"));
    connect(resetAction, &QAction::triggered, this, [this]() {
        m_tableView->reset();
    });
    
    // Включаем печать, если есть контент
    printingEnabled(hasContent());
}

bool CsvViewer::parseFile()
{
    if (!m_file || !m_file->isOpen()) {
        if (m_file)
            m_file->open(QIODevice::ReadOnly | QIODevice::Text);
    }
    
    if (!m_file || !m_file->isOpen())
        return false;
    
    m_model->clear();
    QTextStream stream(m_file.get());
    
    bool firstRow = true;
    int columnCount = 0;
    
    while (!stream.atEnd()) {
        QString line = stream.readLine();
        QStringList fields = line.split(',');
        
        if (firstRow) {
            // Первая строка - заголовки
            columnCount = fields.size();
            m_model->setHorizontalHeaderLabels(fields);
            firstRow = false;
        } else {
            // Данные
            QList<QStandardItem*> items;
            for (const QString &field : fields) {
                items.append(new QStandardItem(field.trimmed()));
            }
            m_model->appendRow(items);
        }
    }
    
    // Сообщаем, что документ загружен
    documentLoaded(m_file->fileName());
    
    return m_model->rowCount() > 0;
}

QByteArray CsvViewer::saveState() const
{
    // Сохраняем состояние (например, ширину колонок)
    QByteArray state;
    QDataStream stream(&state, QIODevice::WriteOnly);
    
    if (m_tableView) {
        // Сохраняем состояние заголовков
        stream << m_tableView->horizontalHeader()->saveState();
    }
    
    return state;
}

bool CsvViewer::restoreState(QByteArray &state)
{
    if (state.isEmpty() || !m_tableView)
        return false;
    
    QDataStream stream(&state, QIODevice::ReadOnly);
    QByteArray headerState;
    stream >> headerState;
    
    return m_tableView->horizontalHeader()->restoreState(headerState);
}

#ifdef QT_DOCUMENTVIEWER_PRINTSUPPORT
void CsvViewer::printDocument(QPrinter *printer) const
{
    if (!hasContent() || !printer)
        return;
    
    // Простая печать таблицы
    QPainter painter(printer);
    
    // Здесь должна быть логика печати таблицы
    // Для простоты опущена
    
    painter.end();
    
    // Уведомляем о статусе печати
    const_cast<CsvViewer*>(this)->printStatusChanged(
        AbstractViewer::PrintStatus::PrintSuccess
    );
}
#endif
```

### Шаг 4: JSON метаданные (csvviewer.json)

```json
{
    "Name": "CsvViewer",
    "Description": "Viewer for CSV (Comma-Separated Values) files",
    "Version": "1.0.0",
    "Author": "Your Name",
    "License": "BSD-3-Clause",
    "MimeTypes": [
        "text/csv",
        "text/comma-separated-values",
        "application/csv"
    ],
    "FileExtensions": [
        "csv",
        "tsv"
    ],
    "Features": [
        "view",
        "print"
    ]
}
```

### Шаг 5: CMakeLists.txt

```cmake
# CSV Viewer Plugin

qt_add_plugin(csvviewer)

target_sources(csvviewer PRIVATE
    csvviewer.cpp
    csvviewer.h
)

target_link_libraries(csvviewer PRIVATE
    Qt6::Core
    Qt6::Gui
    Qt6::Widgets
)

# Опциональная поддержка печати
if(TARGET Qt6::PrintSupport)
    target_link_libraries(csvviewer PRIVATE Qt6::PrintSupport)
endif()

# Устанавливаем плагин в правильную директорию
install(TARGETS csvviewer
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}/plugins
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}/plugins
)
```

### Шаг 6: Регистрация в главном CMakeLists.txt

Отредактируйте `plugins/CMakeLists.txt`:

```cmake
add_subdirectory(txtviewer)
add_subdirectory(imageviewer)
add_subdirectory(jsonviewer)
add_subdirectory(pdfviewer)
add_subdirectory(q3dviewer)
add_subdirectory(csvviewer)  # Добавьте эту строку
```

## 🎨 Работа с UI

### Создание меню

```cpp
void MyViewer::setupUi()
{
    // Получаем главное меню
    QMenuBar *menu = menuBar();
    
    // Создаем собственное меню
    QMenu *myMenu = menu->addMenu(tr("My Menu"));
    
    // Добавляем действия
    QAction *action1 = myMenu->addAction(tr("Action 1"));
    connect(action1, &QAction::triggered, this, &MyViewer::onAction1);
    
    // Действие с иконкой и горячей клавишей
    QAction *action2 = myMenu->addAction(
        QIcon(":/images/icon.png"), 
        tr("Action 2")
    );
    action2->setShortcut(QKeySequence::Refresh);
    connect(action2, &QAction::triggered, this, &MyViewer::onAction2);
}
```

### Создание панели инструментов

```cpp
void MyViewer::setupToolbar()
{
    // Создаем панель инструментов
    QToolBar *toolbar = new QToolBar(tr("My Toolbar"), mainWindow());
    
    // Добавляем действия
    toolbar->addAction(QIcon(":/images/zoom-in.png"), tr("Zoom In"), 
                      this, &MyViewer::zoomIn);
    toolbar->addAction(QIcon(":/images/zoom-out.png"), tr("Zoom Out"), 
                      this, &MyViewer::zoomOut);
    
    toolbar->addSeparator();
    
    // Добавляем виджет (например, комбобокс)
    QComboBox *combo = new QComboBox(toolbar);
    combo->addItems({"Option 1", "Option 2", "Option 3"});
    toolbar->addWidget(combo);
    
    // Добавляем панель в главное окно
    mainWindow()->addToolBar(Qt::TopToolBarArea, toolbar);
    
    // Сохраняем для последующего удаления
    addedToolBars(toolbar);
}
```

### Работа с обзором (Overview)

Если ваш плагин поддерживает обзор (thumbnails, bookmarks):

```cpp
bool MyViewer::supportsOverview() const
{
    return true;  // Включаем поддержку
}

void MyViewer::setupOverview()
{
    // Создаем виджет для закладок
    QListWidget *bookmarks = new QListWidget(parent());
    bookmarks->addItems({"Chapter 1", "Chapter 2", "Chapter 3"});
    
    connect(bookmarks, &QListWidget::itemClicked, 
            this, &MyViewer::onBookmarkClicked);
    
    // Устанавливаем виджет закладок
    setBookmarkWidget(bookmarks);
    
    // Аналогично для миниатюр
    QListView *thumbnails = new QListView(parent());
    // ... настройка thumbnails
    setPageWidget(thumbnails);
}
```

## 💾 Сохранение состояния

```cpp
QByteArray MyViewer::saveState() const
{
    QByteArray data;
    QDataStream stream(&data, QIODevice::WriteOnly);
    
    // Версия формата
    stream << quint32(1);
    
    // Сохраняем настройки
    stream << m_zoomLevel;
    stream << m_currentPage;
    stream << m_viewMode;
    
    // Сохраняем состояние виджетов
    if (m_tableView) {
        stream << m_tableView->horizontalHeader()->saveState();
    }
    
    return data;
}

bool MyViewer::restoreState(QByteArray &state)
{
    if (state.isEmpty())
        return false;
    
    QDataStream stream(&state, QIODevice::ReadOnly);
    
    quint32 version;
    stream >> version;
    
    if (version != 1)
        return false;
    
    // Восстанавливаем настройки
    stream >> m_zoomLevel;
    stream >> m_currentPage;
    stream >> m_viewMode;
    
    // Применяем настройки
    applyZoom(m_zoomLevel);
    gotoPage(m_currentPage);
    
    return true;
}
```

## 🖨️ Печать документов

```cpp
#ifdef QT_DOCUMENTVIEWER_PRINTSUPPORT
void MyViewer::printDocument(QPrinter *printer) const
{
    if (!printer || !hasContent())
        return;
    
    // Уведомляем о начале печати
    const_cast<MyViewer*>(this)->printStatusChanged(
        AbstractViewer::PrintStatus::PrintInProgress
    );
    
    QPainter painter;
    if (!painter.begin(printer)) {
        const_cast<MyViewer*>(this)->printStatusChanged(
            AbstractViewer::PrintStatus::PrintError
        );
        return;
    }
    
    // Логика печати
    for (int page = 0; page < totalPages(); ++page) {
        if (page > 0)
            printer->newPage();
        
        // Рисуем страницу
        drawPage(&painter, page, printer->pageRect());
    }
    
    painter.end();
    
    // Уведомляем об успехе
    const_cast<MyViewer*>(this)->printStatusChanged(
        AbstractViewer::PrintStatus::PrintSuccess
    );
}
#endif
```

## 📚 Лучшие практики

### 1. Управление памятью

```cpp
// ✅ Хорошо: используйте родительские объекты Qt
m_widget = new QWidget(parent());

// ✅ Хорошо: используйте умные указатели для non-Qt объектов
std::unique_ptr<MyClass> m_data;

// ❌ Плохо: ручное управление памятью
MyClass *data = new MyClass();  // Кто удалит?
```

### 2. Обработка ошибок

```cpp
bool MyViewer::openFile()
{
    if (!m_file || !m_file->exists()) {
        statusString(tr("File does not exist"));
        return false;
    }
    
    if (!m_file->open(QIODevice::ReadOnly)) {
        statusString(tr("Cannot open file: %1")
            .arg(m_file->errorString()));
        return false;
    }
    
    // ... обработка файла
    
    return true;
}
```

### 3. Асинхронная загрузка

Для больших файлов используйте асинхронную загрузку:

```cpp
void MyViewer::loadFileAsync()
{
    // Показываем индикатор загрузки
    statusString(tr("Loading..."));
    
    // Загружаем в другом потоке
    QFuture<bool> future = QtConcurrent::run([this]() {
        return parseFile();
    });
    
    // Обрабатываем результат
    auto *watcher = new QFutureWatcher<bool>(this);
    connect(watcher, &QFutureWatcher<bool>::finished, this, [this, watcher]() {
        if (watcher->result()) {
            statusString(tr("Loaded successfully"));
            documentLoaded(m_file->fileName());
        } else {
            statusString(tr("Failed to load file"));
        }
        watcher->deleteLater();
    });
    
    watcher->setFuture(future);
}
```

### 4. Локализация

```cpp
// Используйте tr() для всех пользовательских строк
QString message = tr("File loaded: %1 rows").arg(rowCount);

// Для множественного числа
QString message = tr("%n file(s) found", "", fileCount);
```

### 5. MIME-типы

```cpp
QStringList MyViewer::supportedMimeTypes() const
{
    // Возвращайте все возможные MIME-типы
    return QStringList{
        "text/markdown"_L1,
        "text/x-markdown"_L1,
        "text/plain"_L1  // Fallback
    };
}
```

## 🧪 Тестирование плагина

```bash
# Соберите проект
cmake --build build

# Проверьте, что плагин создан
ls build/plugins/myviewer/

# Запустите с отладкой плагинов
export QT_DEBUG_PLUGINS=1
./build/app/documentviewer

# Откройте тестовый файл
./build/app/documentviewer test.csv
```

## 📖 Дополнительные ресурсы

- [Qt Plugin System](https://doc.qt.io/qt-6/plugins-howto.html)
- [viewerinterfaces.h](../app/viewerinterfaces.h) - определение интерфейса
- [abstractviewer.h](../app/abstractviewer.h) - базовый класс
- Существующие плагины в `plugins/` - примеры реализации

---

**Удачи в разработке плагинов!** 🚀

